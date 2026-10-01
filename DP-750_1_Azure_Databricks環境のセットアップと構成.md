# DP-750: Azure Databricks 環境のセットアップと構成

> **対象資格**: Microsoft Certified: Azure Databricks Data Engineer Associate（試験 DP-750）
> **レベル**: 中級
> **試験ウェイト**: 15〜20%（スキル領域 1/4）
> **元ラーニングパス**: [Set up and configure an Azure Databricks environment](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-set-up-configure-environment/)
> **試験の出題範囲**: [DP-750 スタディガイド](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)（2026-10-19 版スキル）

---

## ラーニングパスの概要

Azure Databricks のワークスペース作成から、アーキテクチャ（コントロールプレーンとコンピュートプレーン）、Microsoft サービスとの統合、コンピュートの選択・構成、そして Unity Catalog のオブジェクト整理までを扱う基礎パス。後続の3パス（セキュリティ・データ処理・デプロイ）の前提となる土台で、ウェイトは小さめだが、**コンピュート種別の選択**と **Unity Catalog のオブジェクト作成・DDL** は具体的な設計判断が問われやすい。

### 前提条件
- データ分析の基礎知識
- クラウドストレージの基本的な理解
- SQL とデータ整理の原則に関する知識

---

## モジュール構成

| # | モジュール名 | 概要 | 主な試験対象 |
|---|---|---|---|
| 1 | Explore Azure Databricks | ワークスペース作成、ワークロード、主要概念、ガバナンス | 背景知識 |
| 2 | Understand Azure Databricks architecture | アカウント階層、プレーン構成、ストレージの種類 | 背景知識（設計判断の前提） |
| 3 | Understand Azure Databricks integrations | Fabric / Power BI / VS Code / Purview / Foundry 等との連携 | 背景知識 |
| 4 | Select and configure compute | コンピュート種別、性能・機能設定、ライブラリ、権限 | **コンピュートの選択と構成** |
| 5 | Create and organize objects in Unity Catalog | カタログ〜テーブルの作成、外部カタログ、Genie 指示 | **Unity Catalog オブジェクトの作成と整理** |

> 試験スキルとして明示されているのはモジュール 4・5 の内容。モジュール 1〜3 は、4・5 の設計判断を理解するための前提知識として押さえる。

---

## モジュール 1: Explore Azure Databricks

### 学習目標
- Azure Databricks ワークスペースをプロビジョニングできる
- Azure Databricks の主要ワークロードを識別できる

### 1-1. ワークロードと主要概念

Azure Databricks は Apache Spark をベースとしたレイクハウス・プラットフォームで、次の用途を1つの基盤で扱う。

| ワークロード | 内容 |
|---|---|
| データエンジニアリング | ETL/ELT パイプライン、バッチ・ストリーミング処理 |
| データウェアハウス / BI | SQL ウェアハウスによる分析クエリ、ダッシュボード |
| 機械学習 / AI | モデル開発、MLflow、AI/BI Genie などの AI 機能 |

**主要概念**: ワークスペース（作業環境）、ノートブック、コンピュート、Delta Lake（ACID トランザクションを備えたテーブル形式）、Unity Catalog（ガバナンス基盤）。

### 1-2. Unity Catalog と Microsoft Purview によるガバナンス

- **Unity Catalog**: Databricks 内のデータ・AI 資産を一元管理するメタストア（アクセス制御、リネージ、監査）
- **Microsoft Purview**: オンプレミス・マルチクラウド・SaaS を横断するデータガバナンス（スキャン、分類、データカタログ）
- 両者は補完関係。Databricks 内の細粒度制御は Unity Catalog、組織横断の発見・分類は Purview。

---

## モジュール 2: Understand Azure Databricks architecture

### 学習目標
- アカウント階層、コントロールプレーンとコンピュートプレーンの違いを説明できる
- 利用可能なストレージの種類を使い分けられる

### 2-1. アカウント階層とプレーン

```
Databricks アカウント
  ├─ Unity Catalog メタストア（リージョンごと）
  └─ ワークスペース（複数）
        └─ カタログ → スキーマ → テーブル / ビュー / ボリューム
```

| プレーン | 所在 | 内容 |
|---|---|---|
| **コントロールプレーン** | Databricks 管理のサブスクリプション | Web UI、ジョブスケジューラ、ノートブック、クラスター管理 |
| **コンピュートプレーン（サーバーレス）** | Databricks 管理 | サーバーレスコンピュート。インフラ管理不要 |
| **コンピュートプレーン（クラシック）** | **顧客の Azure サブスクリプション** | クラシックコンピュート。VNet やネットワーク制御を顧客が持つ |

**設計判断**: ネットワーク分離や既存 VNet の利用が必須ならクラシック、運用負荷の低減と起動の速さを優先するならサーバーレス。

### 2-2. ストレージの種類

| 種類 | 概要 | データ管理 | 典型的な用途 |
|---|---|---|---|
| **Unity Catalog マネージドストレージ** | メタストア／カタログ／スキーマ単位で割り当てる保存場所 | Unity Catalog がデータを管理 | 新規データ |
| **外部ストレージ** | 外部ロケーション（ストレージ資格情報＋クラウドパス）で接続する ADLS Gen2 等 | **利用者がファイルを管理**、UC がアクセスを統制 | 既存・共有データ |
| **既定ストレージ** | サーバーレス向けに Databricks が用意する保存場所 | Databricks 管理 | 手早く始める用途 |

- **ストレージ資格情報**は UC のセキュリティ保護可能オブジェクトで、Azure 側の ID（**マネージド ID** 等）を参照する。
- **外部ロケーション** = ストレージパス + ストレージ資格情報。カタログやスキーマのマネージドストレージ場所としても指定できる。
- マネージドストレージの場所は、**スキーマ > カタログ > メタストア**の順に、より下位の指定が優先される。

---

## モジュール 3: Understand Azure Databricks integrations

### 学習目標
- Azure Databricks と主要な Microsoft サービスの連携方法を説明できる

| 連携先 | 連携の要点 |
|---|---|
| **Microsoft Fabric** | Databricks のカタログを Fabric にミラーリング／OneLake と相互にデータへアクセスし、コピーを最小化 |
| **Power BI** | Databricks コネクタで SQL ウェアハウスに接続（Import / DirectQuery）。UC のアクセス制御と併用 |
| **Visual Studio Code** | Databricks 拡張機能と Databricks Connect でローカル開発・デバッグ |
| **Power Platform / Copilot Studio** | コネクタ経由で Databricks のデータをアプリ・エージェントから利用 |
| **Microsoft Purview** | UC 資産のスキャン・分類・カタログ化 |
| **Microsoft Foundry** | Foundry のエージェントが Databricks の **Genie スペース**に接続し、自然言語でデータに問い合わせ |

---

## モジュール 4: Select and configure compute in Azure Databricks

### 学習目標
- ワークロードに適したコンピュート種別を選択できる
- 性能設定・機能設定・ライブラリ・アクセス権限を構成できる

### 4-1. コンピュート種別の選択

| 種別 | 特徴 | 向いているケース |
|---|---|---|
| **サーバーレス** | インフラ管理不要、高速起動、自動スケール、自動アップグレード | 対話的作業・ジョブ全般の第一候補。運用負荷を下げたいとき |
| **ジョブコンピュート** | ジョブ実行時に作成され、完了で自動終了 | 本番の定期バッチ。**対話用より低コスト** |
| **SQL ウェアハウス** | SQL 分析・BI 向けに最適化（サーバーレス / Pro / クラシック） | BI、アドホック SQL、ダッシュボード |
| **クラシックコンピュート（汎用）** | 顧客サブスクリプション内で起動する、構成自由度の高いクラスター | 特殊なライブラリ・構成・ネットワーク要件があるとき |
| **共有コンピュート** | 複数ユーザーで共有（標準アクセスモード） | 複数人で UC のガバナンスを効かせつつ共用 |

**覚え方**: 「本番バッチ → ジョブコンピュート」「BI/SQL → ウェアハウス」「迷ったらサーバーレス」「ネットワーク/構成の独自要件 → クラシック」。

### 4-2. 性能設定

| 設定 | 要点 |
|---|---|
| ノードタイプ / サイズ | メモリ重視・コンピュート重視・GPU など、ワークロード特性で選択 |
| ワーカー数・**オートスケール** | 最小〜最大ワーカーを設定し、負荷に応じて増減。コストと性能の両立 |
| **自動終了（Termination）** | 非アクティブ時間で停止し、アイドルコストを防ぐ |
| **プール** | アイドルインスタンスを事前確保して起動時間を短縮 |

### 4-3. 機能設定

- **Photon**: ネイティブのベクトル化クエリエンジン。SQL・DataFrame 処理を高速化（DBU 単価は上がるが、総コストが下がることも多い）。
- **Databricks Runtime（Spark バージョン）**: 安定性重視なら **LTS** を選ぶ。ML ワークロードには ML 用ランタイムがある。
- **機械学習**: ML ランタイム（ML ライブラリ同梱）や GPU ノードを選択。

### 4-4. ライブラリのインストール

| スコープ | 方法 |
|---|---|
| クラスター単位 | クラスター設定の Libraries（PyPI / Maven / ボリューム・ワークスペース上の wheel/JAR） |
| ノートブック単位 | `%pip install ...`（そのセッション限定） |
| サーバーレス | クラスターではなく **環境（Environment）** にライブラリ・依存関係を定義 |

> 試験では「全ユーザーに同じライブラリが必要 → クラスターライブラリ」「そのノートブックだけ → ノートブックスコープ」といった切り分けが問われる。

### 4-5. コンピュートへのアクセス権限

- コンピュートのアクセス権限: **CAN ATTACH TO**（アタッチして使用）／ **CAN RESTART**（起動・再起動）／ **CAN MANAGE**（構成変更・権限管理）
- **アクセスモード**: 標準（複数ユーザー共有・UC 対応）／ 専用（単一ユーザーまたはグループに割り当て）
- **クラスターポリシー**で、作成可能な構成（ノードタイプ、最大ワーカー数、自動終了など）を制限し、コストとセキュリティを統制する。

---

## モジュール 5: Create and organize objects in Unity Catalog

### 学習目標
- 要件に応じてカタログ・スキーマ・ボリューム・テーブル・ビューを作成できる
- 外部カタログ、命名規則、Genie 指示を構成できる

### 5-1. 3 階層名前空間と命名規則

```
<catalog>.<schema>.<object>
```

**カタログは分離の単位**として設計するのが一般的。

| 要件 | 命名・分離の例 |
|---|---|
| 環境分離 | `dev_sales` / `test_sales` / `prod_sales` |
| チーム／ドメイン分離 | `finance`、`marketing` |
| 外部共有 | 共有専用カタログを分け、共有対象を限定 |

### 5-2. オブジェクトの作成

```sql
CREATE CATALOG IF NOT EXISTS prod_sales;
CREATE SCHEMA  IF NOT EXISTS prod_sales.bronze;
CREATE VOLUME  IF NOT EXISTS prod_sales.bronze.raw_files;   -- 非構造化／ファイル用
CREATE TABLE   prod_sales.bronze.orders (id BIGINT, amount DECIMAL(10,2));
CREATE VIEW    prod_sales.gold.v_big_orders AS
  SELECT * FROM prod_sales.bronze.orders WHERE amount > 1000;
CREATE MATERIALIZED VIEW prod_sales.gold.mv_daily AS
  SELECT order_date, SUM(amount) AS total FROM prod_sales.bronze.orders GROUP BY order_date;
```

| オブジェクト | 用途 |
|---|---|
| **ボリューム** | ファイル（CSV、画像、モデル等）のガバナンス。**マネージド**／**外部**がある |
| **テーブル** | 構造化データ。**マネージド**／**外部** |
| **ビュー** | クエリの論理定義（データを持たない） |
| **マテリアライズドビュー** | クエリ結果を保持し、更新で最新化（パフォーマンス向上） |

### 5-3. マネージドテーブルと外部テーブルの DDL

| 観点 | マネージド | 外部 |
|---|---|---|
| データの場所 | UC のマネージドストレージ | `LOCATION` で指定した外部ロケーション |
| `DROP TABLE` | **メタデータとデータの両方**が削除対象（保持期間後に削除） | **メタデータのみ**削除。データは残る |
| 既定の推奨 | 新規テーブルの既定 | 既存・共有データの登録 |

```sql
-- 外部テーブル
CREATE TABLE prod_sales.bronze.ext_orders
LOCATION 'abfss://data@account.dfs.core.windows.net/orders';
```

### 5-4. 外部カタログ（Lakehouse Federation）

外部データベース（SQL Server、PostgreSQL 等）を **データを移動せずに**参照する仕組み。

1. **接続（Connection）** を作成（接続先と資格情報を保持）
2. その接続を使って **外部カタログ（foreign catalog）** を作成
3. 外部のスキーマ・テーブルが UC のカタログとして現れ、UC の権限で制御できる

### 5-5. AI/BI Genie 指示

Genie スペースに **指示（instructions）** を設定すると、自然言語での問い合わせ精度が上がり、データ発見性が向上する。

- 用語の定義（例:「売上」はどの列か）
- 対象テーブルの説明・結合関係
- サンプルクエリ、回答時の注意事項

> 良質なテーブル／列コメントが Genie の回答品質の土台になる（ガバナンス領域とも連動）。

---

## まとめ: このスキル領域の重要ポイント

### 設計判断の対応表

| 状況 | 選択 |
|---|---|
| 運用負荷を下げたい、迷った | サーバーレス |
| 定期バッチの本番実行 | ジョブコンピュート |
| BI・SQL 分析 | SQL ウェアハウス |
| 独自ネットワーク・特殊構成 | クラシックコンピュート |
| 起動時間の短縮 | プール |
| アイドルコスト防止 | 自動終了 |
| 環境（dev/prod）の分離 | 環境別カタログ |
| 既存の ADLS データを UC 管理下に | 外部ロケーション + 外部テーブル／ボリューム |
| 外部 DB を移動せず参照 | 接続 → 外部カタログ |

### 試験での重要ポイント
- マネージド vs 外部で **`DROP` 時のデータの扱い**が違う
- マネージドストレージの優先順位は **スキーマ > カタログ > メタストア**
- ライブラリのスコープ（クラスター / ノートブック / サーバーレス環境）の使い分け
- Photon・LTS ランタイム・オートスケール・自動終了の目的

---

## 参考リソース

- [DP-750 スタディガイド](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)
- [ラーニングパス: Set up and configure an Azure Databricks environment](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-set-up-configure-environment/)
- [Azure Databricks ドキュメント](https://learn.microsoft.com/en-us/azure/databricks/)

> **作成上の注記**: 各モジュールの目次・学習目標・試験スキルは Microsoft Learn の公式ページから取得。ユニット本文の細部（UI 操作やコマンドの詳細）は取得していないため、一般的な Azure Databricks の仕様知識で補っている。最新の仕様は公式ドキュメントで確認すること。
