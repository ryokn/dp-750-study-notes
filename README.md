# DP-750 学習ノート

Microsoft Certified: Azure Databricks Data Engineer Associate（試験 DP-750）向けの日本語学習ノート。Microsoft Learn の 4 つのラーニングパスを、試験のスキル領域ごとに要点・比較表・コード例でまとめている。

## ノート一覧

| # | ノート | 試験ウェイト |
|---|---|---|
| 1 | [Azure Databricks 環境のセットアップと構成](DP-750_1_Azure_Databricks環境のセットアップと構成.md) | 15〜20% |
| 2 | [Unity Catalog オブジェクトのセキュリティ保護とガバナンス](DP-750_2_Unity_Catalogオブジェクトのセキュリティ保護とガバナンス.md) | 15〜20% |
| 3 | [データの準備と処理](DP-750_3_データの準備と処理.md) | 30〜35% |
| 4 | [データパイプラインとワークロードのデプロイと保守](DP-750_4_データパイプラインとワークロードのデプロイと保守.md) | 30〜35% |

## 主な内容

- コンピュート種別の選択、Unity Catalog のオブジェクト設計と DDL
- 権限モデル、行フィルター／列マスク、ABAC、リネージ、Delta Sharing
- 取り込み方式（Auto Loader、COPY INTO、CDC など）、SCD、データ品質の期待値
- Lakeflow Jobs、Declarative Automation Bundles による CI/CD、監視と性能チューニング

## 参考

- [DP-750 スタディガイド](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

> 各ノートの細部は一般的な仕様知識で補っている箇所がある。最新の仕様は公式ドキュメントで確認すること。
