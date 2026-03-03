# Liberty Bob Commands

WebSphere Liberty / Open Liberty 開発を支援するBob用スラッシュコマンド集です。

## 概要

このリポジトリは、IBM BobでWebSphere Liberty / Open Libertyの開発・運用を効率化するためのスラッシュコマンドを提供します。コードレビュー、設定管理、トラブルシューティング、移行支援など、日常的な作業を自動化・効率化できます。

## セットアップ

### 1. コマンドの配置

以下のいずれかのディレクトリに`commands/`フォルダを配置してください：

```
.bob/commands/          # プロジェクト固有のコマンド
~/.bob/commands/        # グローバルコマンド
```

### 2. 使い方

Bobのチャット入力欄で `/` を入力すると、利用可能なコマンド一覧が表示されます。

```
/review                 # コードレビューを実行
/liberty-doctor         # Liberty全体の健康診断
```

文字を追加すると候補が絞り込まれます。コマンドを選択後、必要に応じて引数を追加して実行してください。


## スラッシュコマンドとは？

スラッシュコマンドは、チャット入力欄で `/` から始まるコマンドです。よく行う作業（レビュー依頼、手順の定型化、チェックリスト実行など）を短いコマンド1つで呼び出せます。



## コマンド一覧

### 最重要コマンド（日常使用）

#### `/liberty-doctor`
**概要**: Libertyプロジェクト全体を一発健康診断（設定/ビルド/ログ/起動失敗原因を自動チェック）

**使い方**:
```
/liberty-doctor                        # カレントディレクトリを診断
/liberty-doctor path/to/server.xml     # 特定のserver.xmlを診断
```

**主な機能**:
- 設定の散らばり（server.xml、configDropins）を可視化
- ビルドツール・プラグインの整合性チェック
- ポート競合・バインド系の確認
- よくある起動失敗原因の候補化
- feature最小化が必要な場合は`/liberty-feature-min`へ誘導



#### `/liberty-log-triage`
**概要**: Liberty のログを自動トリアージし、原因候補を重要度順に整理・分類

**使い方**:
```
/liberty-log-triage                              # カレントディレクトリのログを解析
/liberty-log-triage wlp/usr/servers/defaultServer  # 特定サーバーのログを解析
```

**主な機能**:
- messages.log/console.log/FFDCの自動収集
- エラーを重要度順にソート
- 原因をカテゴリ別に分類（設定ミス/feature不足/DB/SSL/認証など）
- 次に見るべきファイル・設定箇所を提示
- 最小修正パッチ案の提供



### 設定・構成管理

#### `/liberty-config-explain`
**概要**: server.xmlとconfigDropinsの実効設定を日本語で説明し、優先順位・衝突・危険設定を指摘

**使い方**:
```
/liberty-config-explain                    # 自動探索
/liberty-config-explain path/to/server.xml # 特定のserver.xmlを解析
```

**主な機能**:
- 実効設定の可視化（defaults → server.xml → overrides）
- 設定の衝突・上書き関係の検出
- 危険設定のハイライト（開発用設定の混入、TLS設定の弱さなど）
- 変数（${...}）の解決状況確認



#### `/liberty-feature-min`
**概要**: generated-features.xmlを生成・差分分析し、server.xmlのfeature最小化案を提示

**使い方**:
```
/liberty-feature-min                        # 自動探索・生成
/liberty-feature-min path/to/server.xml     # 特定のserver.xmlを対象
/liberty-feature-min path/to/server.xml --force  # 強制再生成
```

**主な機能**:
- generated-features.xmlの自動生成（Maven/Gradle対応）
- server.xmlとの差分分析
- 削除候補・残すべきfeatureの分類
- 段階的削減プランの提示
- 検証チェックリストの生成



#### `/liberty-env-vars-audit`
**概要**: Liberty構成の${...}変数参照を棚卸しし、未定義・タイポ疑い・環境差分を検出

**使い方**:
```
/liberty-env-vars-audit                    # カレントディレクトリを棚卸し
/liberty-env-vars-audit path/to/server.xml # 特定のserver.xmlを対象
```

**主な機能**:
- 変数参照（${...}）の全列挙
- 未定義変数・タイポ疑いの検出
- local/dev/stg/prod環境差分の可視化
- 秘匿値直書きの検出（値はマスク）
- 環境別テンプレート（.env、server.env）の提案



### 開発支援

#### `/liberty-add-feature`
**概要**: 指定した機能（JPA、REST Clientなど）に必要なLiberty featureと依存関係を自動追加

**使い方**:
```
/liberty-add-feature JPA              # JPA機能を追加
/liberty-add-feature REST Client      # REST Client機能を追加
/liberty-add-feature CDI              # CDI機能を追加
```

**主な機能**:
- Java EE/Jakarta EE世代の自動判定
- 適切なfeatureの選択と追加
- pom.xml/build.gradleへの依存関係追加
- 最小サンプルコードの提供（オプション）
- 検証チェックリストの生成



#### `/liberty-datasource-create`
**概要**: 指定DBへのDataSourceを自動作成（JDBCドライバー追加→Liberty設定→接続チェック）

**使い方**:
```
/liberty-datasource-create postgres localhost:5432 mydb user env:DB_PASSWORD
/liberty-datasource-create mysql db.example.com mydb user prompt
/liberty-datasource-create oracle host:1521 service user env:ORACLE_PASSWORD
```

**引数**:
- `<dbType>`: postgres/mysql/mariadb/db2/oracle/mssql
- `<host[:port]>`: ホスト名とポート（ポート省略時はデフォルト）
- `<dbName>`: データベース名
- `<user>`: ユーザー名
- `<password>`: パスワード（`env:VAR`で環境変数、`prompt`で対話入力）

**主な機能**:
- JDBCドライバーのpom.xml追加
- server.xmlへのlibrary/jdbcDriver/dataSource追加
- 接続チェックの自動実行
- 環境変数参照の推奨



#### `/liberty-endpoint-map`
**概要**: JAX-RSリソースを走査してRESTエンドポイント一覧を生成し、404/405の原因候補を提示

**使い方**:
```
/liberty-endpoint-map           # カレントディレクトリを探索
/liberty-endpoint-map src/      # 特定ディレクトリを探索
```

**主な機能**:
- @Pathアノテーションの自動検出
- エンドポイント一覧（HTTPメソッド、パス、セキュリティ）の生成
- contextRoot/applicationPathの自動判定
- curl/httpieサンプルの生成
- 404/405トラブルシュート候補の提示



### 移行・アップグレード

#### `/liberty-migrate-hints`
**概要**: 旧Java EE/古いLiberty設定を検出し、段階的移行プランと互換性検証ポイントを提示

**使い方**:
```
/liberty-migrate-hints           # カレントディレクトリを解析
/liberty-migrate-hints src/      # 特定ディレクトリを解析
```

**主な機能**:
- javax.*とjakarta.*の混在検出
- 古い依存関係の検出
- ベンダー依存APIの検出
- 段階的移行プラン（段階0〜4）の提示
- 互換性検証チェックリストの生成



#### `/javaee-dd-versionup`
**概要**: デプロイメントディスクリプタ（DD）のバージョン/名前空間を安全に更新

**使い方**:
```
/javaee-dd-versionup                    # 自動判定で更新
/javaee-dd-versionup . javaee8          # Java EE 8に揃える
/javaee-dd-versionup . jakarta9         # Jakarta EE 9に更新
/javaee-dd-versionup . jakarta10        # Jakarta EE 10に更新
```

**主な機能**:
- web.xml、persistence.xml、beans.xmlなどの自動検出
- Java EE/Jakarta EE世代の自動判定
- namespace/schemaLocation/versionの更新
- 危険な変更の抑止（提案のみ）
- 互換性リスクの警告



### セキュリティ

#### `/liberty-security-check`
**概要**: Libertyの認証・認可・SSL設定を初期監査し、弱い設定や認可漏れ候補を検出

**使い方**:
```
/liberty-security-check                    # 自動探索
/liberty-security-check path/to/server.xml # 特定のserver.xmlを監査
```

**主な機能**:
- 認証方式（basic/oidc/jwt/ldap）の検出
- dev用の緩い設定の検出
- SSL/keystore/truststoreの整合性チェック
- 認可漏れしそうなエンドポイント候補の列挙
- 秘匿値直書きの検出（値はマスク）



### 運用・リリース

#### `/liberty-release-check`
**概要**: リリース前の簡易チェックリストを自動生成（dev→prod設定差分/機密値/ログ過剰/スモーク項目）

**使い方**:
```
/liberty-release-check                      # 自動判定
/liberty-release-check config/dev config/prod  # dev/prod構成を指定
```

**主な機能**:
- dev→prod設定差分の抽出（重要度別）
- 機密値直書きの検出
- ログレベル過剰設定の確認
- スモーク/テスト確認項目の生成
- 監視・ヘルスチェックエンドポイントの確認



#### `/liberty-runbook`
**概要**: 現在の設定・ログ・運用情報からRunbook（障害対応手順書）を自動生成・更新

**使い方**:
```
/liberty-runbook                        # RUNBOOK-LIBERTY.mdを生成
/liberty-runbook docs/runbook.md        # 指定パスに生成
```

**主な機能**:
- 環境情報の自動収集
- 標準運用手順の生成（起動/停止/健全性確認）
- トラブルシュート章の自動生成（起動失敗/DB接続/証明書エラー）
- 既存Runbookの更新（手書き領域を保持）
- 秘密情報のマスキング



### CI/CD・インフラ

#### `/liberty-gitlab`
**概要**: GitLab CI/CDとLibertyを連携（ビルド/テスト→generated-features生成→パッケージ化）

**使い方**:
```
/liberty-gitlab                        # 自動探索・セットアップ
/liberty-gitlab path/to/server.xml     # 特定のserver.xmlを対象
```

**主な機能**:
- .gitlab-ci.ymlの自動生成/更新
- Maven/Gradleの自動判定
- generated-features.xmlの生成
- feature差分レポートの生成
- JUnitレポートの統合



### その他の用途

#### `/liberty-mq-jms-setup`
**概要**: IBM MQ接続設定（JMS）を自動点検し、最小構成のserver.xml案を提示

**使い方**:
```
/liberty-mq-jms-setup                    # 自動探索
/liberty-mq-jms-setup path/to/server.xml # 特定のserver.xmlを対象
```

**主な機能**:
- MQ Resource Adapterの設定確認
- CLIENT/BINDINGS接続の判定
- ConnectionFactory/Queueの設定生成
- MDB利用時のActivationSpec設定
- 接続検証チェックリストの提供



#### `/liberty-scaffold-mp-config`
**概要**: MicroProfile Config前提で設定キー一覧を抽出し、環境別テンプレートを生成

**使い方**:
```
/liberty-scaffold-mp-config           # カレントディレクトリを解析
/liberty-scaffold-mp-config src/      # 特定ディレクトリを解析
```

**主な機能**:
- @ConfigPropertyの自動抽出
- 必須/任意/既定値の判定
- bootstrap.propertiesの生成
- dev/stg/prod環境別テンプレートの生成
- .env.example、Kubernetes Secret/ConfigMapテンプレートの生成



## トラブルシューティング

### Bobがスラッシュコマンドを実行しない

Bobがスラッシュコマンドを実行するのではなく、コマンドの内容を実装しようとしてしまう場合は、BobのモードをCodeやAdvancedに変更してからスラッシュコマンドを実行してください。

モードの変更方法：
1. Bobのチャット画面上部でモード選択ボタンをクリック
2. 「💻 Code」または「🛠️ Advanced」モードを選択
3. スラッシュコマンドを実行



## 貢献

バグ報告や機能追加の提案は、Issueまたはプルリクエストでお願いします。



**Note**: 各コマンドの詳細な使い方や引数については、`commands/`ディレクトリ内の各Markdownファイルを参照してください。