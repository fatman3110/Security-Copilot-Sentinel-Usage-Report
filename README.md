# Sentinel 使用量・コスト レポート

Microsoft Sentinel（Data Lake 対応ワークスペース）の使用量・コストを **オフライン HTML レポート**として定期生成するモジュール。
数値は KQLで算出し、結果やトレンドの分析については **Security Copilot Build Agent** が生成する。

## 前提条件（必要な製品・ライセンス）

| 製品 / サービス | 用途 | 備考 |
|---|---|---|
| **Microsoft Sentinel**（Data Lake 対応ワークスペース） | 使用量・コストの集計対象。`Usage` テーブルおよび Data Lake tier の参照元 | Data Lake へのオンボードが必要。Analytics / Data Lake Only の tier 判定を行う |
| **Microsoft Defender ポータル**（統合 SOC） | Watchlist 登録・運用、コスト管理画面の参照 | Sentinel は Defender ポータルに統合済み |
| **Microsoft Security Copilot** | 注意・アクション コメントの生成（Build Agent 実行） | SCU（Security Compute Unit）のプロビジョニングが必要 |
| **Azure Logic Apps**（従量課金） | レポート生成のオーケストレーション（スケジュール実行） | マネージド ID で Storage / Monitor / Security Copilot に接続 |
| **Azure Storage**（汎用 v2） | テンプレート配置先・レポート出力先 | ARM テンプレートが新規作成。Entra 認証専用（SharedKey 無効） |

> RBAC: デプロイ実行者には対象サブスクリプション／リソースグループへの **Contributor**（リソース作成）と、
> Storage への **Storage Blob Data Contributor**（テンプレート アップロード）が必要。
> Watchlist 登録・Agent 登録には Sentinel / Security Copilot 側の相応の権限が必要。

## 本リポジトリの構成ファイル

| ファイル | 役割 |
|---|---|
| [UsageCostReportAgent.yaml](UsageCostReportAgent.yaml) | Security Copilot のAgent 定義 |
| [logicapp-usage-cost-report.json](logicapp-usage-cost-report.json) | ARM テンプレート。入出力用 Storage/RBAC/接続/Logic App を作成。以降 Logic App でレポートを生成（KQL集計 → Agent → テンプレ差込 → Blob保存） |
| [template.html](template.html) | 出力HTMLレポートのテンプレートファイル |

## データフロー

```
Logic App（スケジュール実行）
 ├─ KQL_usage_aggregate   Usage 集計（GB/前期比）+ lastSeen + _GetWatchlist('TableTiers') で Tier 付与  ← 決定論・数値ソース
 ├─ SecurityCopilot_insights   集計 JSON を Security Copilot Agent へ送り分析
 ├─ Normalize_agent_text  応答からコードフェンス・CR を除去（ラベル抽出用に正規化）
 ├─ Compose_safe_insights ラベル付き応答（STATUS/SURGE/DEAD/TIER/ERROR）から示唆を抽出。欠落時はフォールバック文を充当（数値は影響なし）
 ├─ Select_detail_rows    明細を決定論で生成（数値は KQL のままでLLMによる修正は行わせない）
 ├─ Compose_html          テンプレに数値（決定論）+ 分析による示唆（LLM）を差し込み
 └─ Create_blob_report    usage-report-YYYY-MM-DD.html を Blob 保存
```

## 展開手順

### Phase 1: 事前作業

Data Lake Only Ingest のテーブルデータをwatch list として登録

```powershell
# 1) 対象ワークスペースの値を実値に設定（例の値は自組織のものに置換）
$sub = "<your subscription id>"   # サブスクリプション ID（例: 00000000-0000-0000-0000-000000000000）
$rg  = "<your sentinel resource group>"    # リソースグループ名（例: sentinel-japaneast）
$ws  = "<your sentinel workspace name>"    # ワークスペース名（例: Sentinel）

az login
az account set --subscription $sub

# 2) データ取得＋CSV 生成
$wsId = "/subscriptions/$sub/resourceGroups/$rg/providers/Microsoft.OperationalInsights/workspaces/$ws"
$json = az rest --method get --url "https://management.azure.com$wsId/tables?api-version=2025-07-01" -o json
$r = $json | ConvertFrom-Json
$r.value | Where-Object { $_.properties.plan -eq 'Auxiliary' } |
  ForEach-Object { [pscustomobject]@{ TableName = $_.name; Tier = 'DataLakeOnly' } } |
  Export-Csv -NoTypeInformation -Encoding UTF8 .\TableTiers.csv
```

生成した `TableTiers.csv`（列 `TableName`, `Tier`）をレビュー（誤分類の除外）したうえで、エイリアス `TableTiers` で Watchlist として取り込む。

**Defender ポータル** — 最新手順は公式手順を参照ください [Upload a watchlist from a file you created](https://learn.microsoft.com/azure/sentinel/watchlists-create#upload-a-watchlist-from-a-local-folder) 

1. [Defender ポータル](https://security.microsoft.com/) > **Microsoft Sentinel** > **構成** > **ウォッチリスト**。
2. **+ New** を選択してウィザードを開く。
3. **全般** タブ: 名前 / 説明 を入力し、**エイリアス** に `TableTiers` を入力 → **Next: Source**。
4. **ソース** タブ: 送信元の種類 =`ローカルファイル`、File type=`ヘッダー付き CSV ファイル (.csv)`、見出し行の前の行数=`0`、`TableTiers.csv` をアップロード、SearchKey=`TableName`→ **Next: Review + create**。
5. 内容を確認して **作成**。

### Phase 2: モジュールのデプロイ

**A. ARM テンプレートのデプロイ** — 最新手順は公式手順を参照ください [Deploy resources from custom template](https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-portal#deploy-resources-from-custom-template)

1. [Azure Portal](https://portal.azure.com/) の検索バーで **Deploy a custom template**（カスタム テンプレートのデプロイ）を検索して選択。
2. **Build your own template in the editor**（エディターで独自のテンプレートを作成する）を選択。
3. **ファイルの読み込み** で [logicapp-usage-cost-report.json](logicapp-usage-cost-report.json) を読み込む（または内容を貼り付け）→ **保存**。
4. パラメータ入力画面で **リソースグループ** を選び、各パラメータに**自組織のニーズに応じた値を入力する**（下記の必須／推奨項目を参照）→ **確認と作成**。
   - `sentinelSubscriptionId` / `sentinelResourceGroupName` / `sentinelWorkspaceName`: 集計対象の Sentinel（Log Analytics）ワークスペースを指定（**必須・既定なし**）。
   - `reportStorageAccountName`: 新規作成するストレージ名。グローバル一意・3-24文字英小数字（**必須・既定なし**）。
   - `priceAnalyticsIngestPerGB` / `priceLakeIngestPerGB`: 既定は Japan East 定価。**EA/MCA の割引後実単価がある場合は必ず上書き**する（[IaC パラメータ](#iac-パラメータ)参照）。
   - `recurrenceMonthDay` / `recurrenceHour` / `recurrenceMinute` / `recurrenceTimeZone`: レポート配信タイミングを運用に合わせて調整。
   - `showCost`: 金額を出さず GB のみ表示にしたい場合は `0`。
   - `sentinelRegionLabel`: 対象ワークスペースのリージョンに合わせる（既定 `Japan East`）。
   - その他の既定値で問題なければそのままで可。
5. 検証通過後 **作成**。必要なリソースが一括作成される。

> パラメータの全一覧と各既定値の根拠は下記「[IaC パラメータ](#iac-パラメータ)」を参照。

**B. テンプレート HTML のアップロード** — 最新手順は公式手順を参照ください [Upload a block blob](https://learn.microsoft.com/azure/storage/blobs/storage-quickstart-blobs-portal#upload-a-block-blob)

1. Portal で作成したストレージアカウント > **コンテナー** > `templates` コンテナを開く。
2. **アップロード** を選択し、[template.html](template.html) を選んでアップロード（Blob 名は `template.html`）。

> アップロード操作を行うユーザは対象ストレージへの **ストレージ BLOB データ共同作成者**（または 所有者）が必要

**C. コネクタの認可**

1. 作成された Logic App la-sentinel-usage-report > **API 接続**で `azuremonitorlogs` / `securitycopilot`  > API 接続の編集 を開き、**Authorize（承認する）** で認可を完了したあと、**保存**。

**D. Build Agent の登録** — 最新手順は公式手順を参照ください [Build Security Copilot agents by uploading a YAML](https://learn.microsoft.com/copilot/security/developer/build-agent-yaml-file#steps-to-build-and-upload-the-yaml)

1. [Security Copilot](https://securitycopilot.microsoft.com/) を開き、**ビルド** メニューで **YAML マニフェストのアップロード** 配下の **アップロード** を選択し、[UsageCostReportAgent.yaml](UsageCostReportAgent.yaml) をアップロード。
3. アップロード後、エージェントビルダーにマニフェストの構成（Tools / エージェント概要）が表示される。
4. 内容を確認し（必要なら概要などを調整）、エージェントを発行する。
5. **エージェント** メニューで表示されている **Usage Cost Report Insight Agent** の設定からサインインを行う。

### Phase 3: 通常運用（スケジュール実行）

設定された頻度（デフォルトでは月次）でデータを集計・レポート生成し、Blob Storage に出力する

## 技術仕様
### IaC パラメータ

| param | 既定 | 説明 |
|---|---|---|
| `sentinelSubscriptionId` | （必須・既定なし） | 対象 Sentinel ワークスペースのサブスクリプション ID。デプロイ環境に合わせて指定 |
| `sentinelResourceGroupName` | （必須・既定なし） | 対象 Sentinel ワークスペースのリソースグループ名。デプロイ環境に合わせて指定 |
| `sentinelWorkspaceName` | （必須・既定なし） | 対象の Log Analytics（Sentinel）ワークスペース名。デプロイ環境に合わせて指定 |
| `reportStorageAccountName` | （必須・既定なし） | 新規作成するストレージ名（グローバル一意、3-24文字英小数字） |
| `logicAppName` | `la-sentinel-usage-report` | 作成する Logic App 名 |
| `location` | `[resourceGroup().location]` | リソース作成先リージョン |
| `showCost` | `1` | `0`=GB のみ / `1`=金額表示。金額は既定で **USD** 表示。`usdToJpyRate` に 1 以上を設定すると **JPY** 換算表示になる。既定は Japan East 定価で金額表示 |
| `priceAnalyticsIngestPerGB` | `"6.24"` | Analytics インジェスト単価 **USD/GB**。既定は **Japan East の Pay-as-you-go 定価**（$6.24/GB、Azure Retail Prices API）。事前コミットなどの割引があれば上書き |
| `priceLakeIngestPerGB` | `"0.2175"` | Data Lake Only 取込単価 **USD/GB**。Data Lake への取込は **Data lake ingestion と Data processing が必ず両方課金**されるためその合算。Japan East 定価 = $0.0725 + $0.145 = **$0.2175/GB** |
| `usdToJpyRate` | `158` | **オプション**。ドル円為替レート（例 `150` = 1 USD = 150 円）。**1 以上**を設定するとレポート内の金額をすべて USD から **JPY 換算**して表示。`0` を指定すると USD 表示のまま |
| `lookbackDays` | `30` | 直近期間日数（前期比は直前の同日数） |
| `watchlistAlias` | `TableTiers` | Data Lake Only 登録 Watchlist のエイリアス |
| `recurrenceFrequency` / `recurrenceInterval` | `Month` / `1` | 生成頻度（`Day`/`Week`/`Month`）と間隔 |
| `recurrenceTimeZone` | `Tokyo Standard Time` | トリガーのタイムゾーン（Windows TZ 名） |
| `recurrenceMonthDay` | `1` | **Month 時の実行日**。1=毎月 1 日 / -1=月末。Day/Week では無視 |
| `recurrenceHour` / `recurrenceMinute` | `9` / `0` | 実行時刻（`recurrenceTimeZone` 基準）。既定 09:00 |
| `storageSkuName` | `Standard_LRS` | ストレージ SKU |
| `reportContainerName` | `usage-reports` | レポート出力コンテナ |
| `templateContainerName` / `templateBlobName` | `templates` / `template.html` | テンプレート配置先 |
| `sentinelRegionLabel` | `Japan East` | レポート表示用リージョンラベル（既定単価が Japan East のため整合） |
| `outputLanguage` | `Japanese` | レポート／示唆の言語（`English` / `Japanese`） |

> `sentinelSubscriptionId` / `sentinelResourceGroupName` / `sentinelWorkspaceName` / `reportStorageAccountName` は**既定値を持たない必須パラメータ**。デプロイ時に自組織の値を必ず指定すること。

> 既定（`showCost=1`）で **Japan East の Pay-as-you-go 定価**で金額を表示する。
> - Analytics tier: **$6.24/GB**
> - Data Lake Only: **$0.2175/GB** = Data lake ingestion $0.0725 + Data processing $0.145（Data Lake 取込は両メーターが必ず課金されるため合算）
> 
> 自組織の実単価を反映したい場合は `priceAnalyticsIngestPerGB` / `priceLakeIngestPerGB` を上書きする。金額を出したくなければ `showCost=0` で GB のみ表示に戻す。

### ウォッチリスト `TableTiers` 仕様

Data Lake Only テーブルの識別に Watchlist を使う。**Data Lake Only テーブルのみ**を登録する（Analytics は登録しない）。

| 列 | 内容 | 例 |
|---|---|---|
| `TableName` | テーブル名（Usage の DataType と一致） | `DeviceLogonEvents` |
| `Tier` | 固定値 `DataLakeOnly` | `DataLakeOnly` |

KQL 側で Watchlist に存在するテーブルを `LakeOnly`、それ以外を `Analytics` と分類する。

### HTML 生成ロジック

Logic App が以下を決定論で合成する（数値は KQL、示唆は耐障害版 Agent 出力）:

```
Read_template        templates/template.html を Blob から読込（マネージド ID 認証）
KQL_summary          ① tier 別合計 GB・前期比・件数を集計
Select_chart_rows    ② Top10 棒グラフ <div> を生成（幅 = 最大値比）
Select_detail_rows   ④ 明細 <tr> を生成
Compose_html         テンプレの {{TOKEN}} 21 個を replace() 連鎖で実値に差替
Create_blob_report   usage-reports/usage-report-YYYY-MM-DD.html を保存
```

テンプレ（[template.html](template.html)）を差し替えるだけでデザイン変更でき、Logic App の再デプロイは不要。

## ライセンス

 [MIT License](LICENSE)