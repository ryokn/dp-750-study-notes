# DP-750: Unity Catalog オブジェクトのセキュリティ保護とガバナンス

> **対象資格**: Microsoft Certified: Azure Databricks Data Engineer Associate（試験 DP-750）
> **レベル**: 中級
> **試験ウェイト**: 15〜20%（スキル領域 2/4）
> **元ラーニングパス**: [Secure and govern Unity Catalog objects in Azure Databricks](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-secure-govern-unity-catalog/)
> **試験の出題範囲**: [DP-750 スタディガイド](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)（2026-10-19 版スキル）

---

## ラーニングパスの概要

Unity Catalog による**アクセス制御**（権限、行・列レベル制御、資格情報管理）と**ガバナンス**（説明、ABAC、リネージ、監査、Delta Sharing）を扱う。認証まわり（Key Vault、サービスプリンシパル、マネージド ID）は Azure 側の知識（Microsoft Entra ID）と組み合わせて問われるため、「どの ID をどの場面で使うか」の判断が重要。

### 前提条件
- Azure Databricks ワークスペースと Unity Catalog の理解
- SQL とデータアクセスパターンの知識
- Microsoft Entra ID と Azure セキュリティの基礎

---

## モジュール構成

| # | モジュール名 | 概要 |
|---|---|---|
| 1 | Secure Unity Catalog objects | 権限付与、行・列レベル制御、Key Vault、サービスプリンシパル、マネージド ID |
| 2 | Govern Unity Catalog objects | 説明・メタデータ、ABAC、行フィルター／列マスク、保持、リネージ、監査、Delta Sharing |

---

## モジュール 1: Secure Unity Catalog objects

### 学習目標
- プリンシパルにセキュリティ保護可能オブジェクトへの権限を付与できる
- テーブル／列レベルのアクセス制御と行レベルセキュリティを実装できる
- Key Vault のシークレットを使い、サービスプリンシパルとマネージド ID で認証できる

### 1-1. 権限モデル

Unity Catalog は ANSI SQL の DCL（`GRANT` / `REVOKE`）に準拠する。

- **プリンシパル**: ユーザー、**サービスプリンシパル**（プログラム用 ID）、**グループ**
- **セキュリティ保護可能オブジェクト**: メタストア、カタログ、スキーマ、テーブル、ビュー、ボリューム、関数、外部ロケーション、ストレージ資格情報 など
- **既定は拒否**（明示的に付与しない限りアクセス不可）

```sql
GRANT USE CATALOG ON CATALOG corpdata TO `analysts`;
GRANT USE SCHEMA  ON SCHEMA  corpdata.sales TO `analysts`;
GRANT SELECT      ON TABLE   corpdata.sales.orders TO `analysts`;
```

**テーブルを読むのに最低限必要な権限**

| 階層 | 必要な権限 |
|---|---|
| カタログ | `USE CATALOG` |
| スキーマ | `USE SCHEMA` |
| テーブル | `SELECT` |

- **継承**: 上位（カタログ／スキーマ）で付与した権限は、**将来作成されるオブジェクトも含めて**配下に継承される。
- 運用の基本は **個人ではなくグループに付与**し、最小権限の原則に従う。
- 付与は SQL のほか、**Catalog Explorer** の UI からも可能。

> **試験のポイント**: 「`SELECT` だけでは読めない」。`USE CATALOG` と `USE SCHEMA` が揃って初めてクエリできる。

### 1-2. テーブル／列レベルのアクセス制御と行レベルセキュリティ

アクセス制御は4つの考え方が組み合わさる（権限、ビュー、行フィルター／列マスク、ABAC）。

| 手法 | 仕組み | 使いどころ |
|---|---|---|
| **テーブル／スキーマ権限** | `GRANT` で粒度を制御 | 基本の制御 |
| **ビュー（動的ビュー）** | 公開する列・行をビューで絞り、ビューにだけ権限を付与 | 列の公開範囲を限定したい |
| **行フィルター** | 関数でユーザー／グループごとに見える行を制限 | 行レベルセキュリティ |
| **列マスク** | 関数で列値をマスク（例: メール、カード番号） | 機微情報の秘匿 |

```sql
-- 行フィルター関数とテーブルへの適用
CREATE FUNCTION sales.region_filter(region STRING)
RETURN IS_ACCOUNT_GROUP_MEMBER('admins') OR region = 'APAC';

ALTER TABLE sales.orders SET ROW FILTER sales.region_filter ON (region);

-- 列マスク関数と適用
CREATE FUNCTION sales.mask_email(email STRING)
RETURN CASE WHEN IS_ACCOUNT_GROUP_MEMBER('pii_readers') THEN email ELSE '***' END;

ALTER TABLE sales.customers ALTER COLUMN email SET MASK sales.mask_email;
```

### 1-3. Azure Key Vault のシークレットへのアクセス

資格情報やパスワードを**ノートブックに直書きしない**。

- Azure Key Vault を**バックエンドとするシークレットスコープ**を作成し、Key Vault のシークレットを Databricks から参照する
- コード内では `dbutils.secrets.get(scope="kv-scope", key="db-password")` で取得（出力時は `[REDACTED]` と表示され、値は漏れにくい）
- スコープへのアクセスは権限（ACL）で制御

### 1-4. サービスプリンシパルによるデータアクセスの認証

- **サービスプリンシパル**: 人ではなく自動化（ジョブ、CI/CD、アプリ）のための ID。Microsoft Entra ID で作成し、Databricks に追加できる。
- 本番ジョブは**個人のユーザーではなくサービスプリンシパルで実行**する（担当者の退職・権限変更の影響を受けない）。
- 認証は OAuth（クライアント ID + シークレット）。シークレットは **Key Vault に保管**する。
- UC 上の権限は、サービスプリンシパルに `GRANT` する。

### 1-5. マネージド ID によるリソースアクセスの認証

- **マネージド ID**: Azure が資格情報を管理する ID。**シークレットのローテーション不要**。
- Azure Databricks では **Azure Databricks 用アクセスコネクタ**（マネージド ID を持つリソース）を使い、**ストレージ資格情報**を作成する。
- 手順の流れ: アクセスコネクタ作成 → ストレージアカウントに **Storage Blob Data Contributor** 等のロールを付与 → UC でストレージ資格情報 → 外部ロケーション作成。

#### サービスプリンシパル vs マネージド ID

| 観点 | サービスプリンシパル | マネージド ID |
|---|---|---|
| 資格情報 | クライアントシークレット／証明書（自分で管理・更新） | **Azure が管理**（シークレットなし） |
| 主な用途 | ジョブ・CI/CD・外部アプリの実行 ID | **ストレージ等の Azure リソースへの接続**（外部ロケーション） |
| 運用負荷 | ローテーションが必要 | 低い |
| 迷ったら | 自動化の「実行主体」 | Azure リソースへの「接続手段」 |

---

## モジュール 2: Govern Unity Catalog objects

### 学習目標
- テーブル・列の定義と説明を整備してデータ発見性を高められる
- ABAC、行フィルター／列マスク、保持ポリシーを構成できる
- リネージ追跡、監査ログ、Delta Sharing を設計・実装できる

### 2-1. 定義と説明によるデータ発見性

```sql
COMMENT ON TABLE sales.orders IS '受注明細。1行=1注文明細';
ALTER TABLE sales.orders ALTER COLUMN amount COMMENT '税抜金額（円）';
```

- テーブル／列のコメントは検索性を高め、**AI/BI Genie** の回答品質にも効く
- **タグ**で分類（機微度、ドメイン、所有者など）を付与できる
- Catalog Explorer では AI 生成コメントの提案も利用できる

### 2-2. 属性ベースアクセス制御（ABAC）

- **タグ**（例: `pii=true`）とポリシーを組み合わせ、**タグに基づいて**アクセス制御を適用する
- オブジェクトごとに個別設定せず、**タグを付けるだけでポリシーが適用される**ためスケールしやすい
- 行フィルター／列マスクの関数を**ポリシーとして**カタログ・スキーマ・テーブルに適用できる

| 方式 | 設定単位 | 向く規模 |
|---|---|---|
| 手動の行フィルター／列マスク | テーブル・列ごと | 少数のテーブル |
| **ABAC** | タグ + ポリシー | 多数のテーブルに一貫して適用 |

### 2-3. 行フィルターと列マスク（再確認）

- 実体は **SQL ユーザー定義関数**。`ALTER TABLE ... SET ROW FILTER` / `ALTER COLUMN ... SET MASK` で適用
- 呼び出したユーザーの所属グループ（`IS_ACCOUNT_GROUP_MEMBER`）で結果を切り替える

### 2-4. データ保持ポリシー

- Delta テーブルの**履歴（ログ）保持**と**削除済みファイルの保持**を設定できる（例: `delta.logRetentionDuration`、`delta.deletedFileRetentionDuration`）
- `VACUUM` で保持期間を過ぎた不要ファイルを物理削除（**保持期間を短くするとタイムトラベルできる範囲も縮む**）
- 法令・社内規程で決まる保持期間とストレージコストのバランスで設計する

### 2-5. データリネージ

- UC は**テーブル・列レベルのリネージ**を自動で収集する
- **Catalog Explorer** のリネージタブで、**上流（ソース）・下流（利用先）**、ノートブック／ジョブ／ダッシュボードなどの依存関係を確認
- 併せて、所有者（owner）、履歴（history）、依存関係（dependencies）も確認できる
- 用途: **影響分析**（列を変更したら何が壊れるか）、データ品質問題の**原因追跡**

### 2-6. 監査ログ

- UC のアクセスや操作は監査ログとして記録される（**システムテーブル**、および Azure の診断設定で Log Analytics 等へ送信）
- 「誰が・いつ・どのデータにアクセスしたか」を追跡し、コンプライアンス要件に対応

### 2-7. Delta Sharing の安全な戦略

**Delta Sharing** は、データをコピーせずに組織外・組織内の相手とライブデータを共有するオープンプロトコル。

| 用語 | 意味 |
|---|---|
| **共有（Share）** | 共有するテーブル等の集合 |
| **受信者（Recipient）** | 共有を受け取る相手 |
| **プロバイダー** | 共有元 |

| 共有方式 | 受信者 | 備考 |
|---|---|---|
| **Databricks 間** | UC 対応の Databricks ワークスペース | 資格情報の受け渡し不要、制御しやすい |
| **オープン共有** | 任意のクライアント（Spark、Power BI、pandas 等） | トークン／資格情報ファイルで接続 |

**安全な設計の要点**
- **共有対象を最小限**に（必要なテーブル・パーティションのみ）
- 受信者ごとに共有を分け、**最小権限**で付与
- オープン共有では**トークンの有効期限**、IP アクセスリスト等で保護
- 共有利用状況を**監査ログで継続的に確認**
- 外部共有用の**専用カタログ**を設けて境界を明確にする（領域 1 の命名・分離と連動）

---

## まとめ: このスキル領域の重要ポイント

### 要件と手段の対応

| 要件 | 手段 |
|---|---|
| チームにテーブルを読ませる | グループへ `USE CATALOG` + `USE SCHEMA` + `SELECT` |
| 地域ごとに見える行を変える | 行フィルター |
| メール等の値を一部ユーザーに隠す | 列マスク |
| 多数テーブルへ一貫した制御 | タグ + ABAC |
| パスワード・キーを安全に扱う | Key Vault バックエンドのシークレットスコープ |
| 本番ジョブの実行 ID | サービスプリンシパル |
| ストレージへのシークレットレス接続 | マネージド ID（アクセスコネクタ + ストレージ資格情報） |
| 影響分析・原因追跡 | リネージ（Catalog Explorer） |
| 誰がアクセスしたかの証跡 | 監査ログ（システムテーブル／Log Analytics） |
| 外部へのデータ提供 | Delta Sharing（最小共有・トークン管理） |

### 試験での重要ポイント
- 読み取りに必要な権限セット（`USE CATALOG` + `USE SCHEMA` + `SELECT`）
- 権限は**継承**され、グループへの付与が推奨
- サービスプリンシパル（自動化の実行主体）と、マネージド ID（Azure リソース接続）の使い分け
- 行フィルター／列マスクは関数ベース、ABAC はタグ + ポリシーで大規模適用
- ABAC・リネージ・Delta Sharing は 2026-10-19 版スキルに含まれる主要トピック

---

## 参考リソース

- [DP-750 スタディガイド](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)
- [ラーニングパス: Secure and govern Unity Catalog objects](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-secure-govern-unity-catalog/)
- [モジュール: Secure Unity Catalog objects](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/)
- [モジュール: Govern Unity Catalog objects](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/)
- [Azure Databricks ドキュメント](https://learn.microsoft.com/en-us/azure/databricks/) / [Microsoft Entra ドキュメント](https://learn.microsoft.com/en-us/entra/)

> **作成上の注記**: 構成・学習目標・試験スキルは Microsoft Learn の公式ページから取得。ユニット本文の細部（ABAC のポリシー構文、各プロパティ名など）は取得しておらず、一般的な仕様知識で補っているため、実装時は最新の公式ドキュメントで確認すること。
