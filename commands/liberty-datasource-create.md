---
description: WebSphere Liberty / Open Liberty の DataSource を作成（pom.xml に JDBC ドライバー追加→Liberty へコピー→server.xml に library/jdbcDriver/dataSource 追加→接続チェック）
argument-hint: "<dbType> <host[:port]> <dbName> <user> <password|env:VAR|prompt> [任意] <server.xmlパス>"
---

あなたは WebSphere Liberty / Open Liberty のアーキテクト兼ビルドエンジニアです。  
目的は「指定 DB へ接続できる DataSource を、最小の設定で安全に自動生成」することです。

本コマンドは次を **自動で実行**します：

1.  DB種別に応じた JDBC ドライバーを `pom.xml` に追加
2.  JDBC ドライバー JAR を Liberty の構成配下へコピーする設定を `pom.xml` に追加
3.  コピーした JAR を参照する `<library>` を `server.xml` に追加
4.  `<jdbcDriver>` と `<dataSource>`（および必要なら `<authData>`）を `server.xml` に追加
5.  接続チェック（JDBC 直叩きの疎通）を実行し結果を提示

> 重要：パスワードをログや差分出力にそのまま表示しない。  
> `env:VAR` 指定があれば **環境変数参照**で `server.xml` に埋め込む。

***

# 入力（引数仕様：オプション最小）

    /liberty-datasource-create <dbType> <host[:port]> <dbName> <user> <password|env:VAR|prompt> [任意] <server.xmlパス>

*   `dbType`（必須）：以下のいずれか（大小文字/ハイフンは許容して正規化）
    *   `postgres` / `postgresql`
    *   `mysql`
    *   `mariadb`
    *   `db2`
    *   `oracle`
    *   `mssql` / `sqlserver`
*   `host[:port]`（必須）：例 `db.example.com:5432` / `localhost`
    *   `:port` が無い場合は DB 種別のデフォルトポートを採用
*   `dbName`（必須）：DB名（サービス名/スキーマ名ではなく DB 名を想定）
*   `user`（必須）
*   `password|env:VAR|prompt`（必須）
    *   `env:DB_PASSWORD` のように指定されたら `server.xml` は `${env.DB_PASSWORD}` を使用
    *   `prompt` の場合は **対話的に入力**（入力値は表示しない想定）
*   `[任意] server.xmlパス`：指定がなければ自動探索。複数見つかった場合だけ最小限質問。

***

# 自動判定ルール（質問しない）

## A) プロジェクト判定

*   ルート探索で `pom.xml` があれば **Maven** とみなす（このコマンドは Maven 前提で進める）
*   `pom.xml` が無い場合は中断し、見つからなかった旨と候補パスを提示

## B) server.xml の決定

*   引数に `server.xmlパス` があればそれを使用
*   なければ自動探索（優先順）
    1.  `src/main/liberty/config/server.xml`
    2.  `config/server.xml`
    3.  `wlp/usr/servers/*/server.xml`
    4.  その他 `server.xml`
*   複数見つかった場合は候補一覧を出し、**どれを編集するかだけ**質問する

***

# DB ごとの生成内容（固定マッピング）

## 1) JDBC ドライバー依存（pom.xml に入れる座標）

*   PostgreSQL: `org.postgresql:postgresql`
*   MySQL: `com.mysql:mysql-connector-j`
*   MariaDB: `org.mariadb.jdbc:mariadb-java-client`
*   DB2: `com.ibm.db2:jcc`
*   Oracle: `com.oracle.database.jdbc:ojdbc11`（Java 11+ 想定）
*   SQL Server: `com.microsoft.sqlserver:mssql-jdbc`

> 既に同一 `groupId:artifactId` が存在する場合は **追加しない**（バージョン上書きもしない）。  
> バージョンが `dependencyManagement` で管理されている場合は `version` を付けない。

## 2) 既定ポート

*   postgres 5432
*   mysql 3306
*   mariadb 3306
*   db2 50000
*   oracle 1521
*   mssql 1433

## 3) JDBC URL テンプレ

*   postgres: `jdbc:postgresql://{host}:{port}/{dbName}`
*   mysql: `jdbc:mysql://{host}:{port}/{dbName}`
*   mariadb: `jdbc:mariadb://{host}:{port}/{dbName}`
*   db2: `jdbc:db2://{host}:{port}/{dbName}`
*   oracle: `jdbc:oracle:thin:@//{host}:{port}/{dbName}`（dbName を serviceName として扱う）
*   mssql: `jdbc:sqlserver://{host}:{port};databaseName={dbName}`

***

# 実行手順（このスラッシュコマンドが行うこと）

## Step 1) 引数を解析し、最小限整形

*   `host[:port]` を分解し、port 未指定なら既定ポートを適用
*   `password` が
    *   `env:VAR` → passwordExpr を `${env.VAR}` にする
    *   `prompt` → 対話入力（非表示）を受け取り、**出力には一切出さない**
    *   それ以外 → 平文扱い。ただし出力表示・差分にはマスク `********` を使用

> 平文 password を server.xml に直書きするのは避けたいので、可能なら `env:` を案内する。ただし本コマンドは要求通り自動で進める（止めない）。

## Step 2) `pom.xml` を更新（JDBC ドライバーを追加）

*   ルートから `pom.xml` を開く
*   `<dependencies>` 内に、上記マッピングの `<dependency>` を追加
    *   `dependencyManagement` で管理されていそうなら version は付けない
    *   管理が無い場合は、`<properties>` に `jdbc.driver.version` を追加し、
        依存には `<version>${jdbc.driver.version}</version>` 形式で入れる（**バージョンはここでは決め打ちしない**）
    *   既に同じ依存があればスキップ

### 追加する例（version 管理が無い場合）

```xml
<properties>
  <!-- JDBC driver version used by /liberty-datasource-create -->
  <jdbc.driver.version>PLEASE_SET_VERSION</jdbc.driver.version>
</properties>

<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <version>${jdbc.driver.version}</version>
</dependency>
```

## Step 3) JDBC ドライバーを Liberty 構成配下へコピーする設定（pom.xml）

*   目的：`src/main/liberty/config/resources/jdbc/` に driver JAR をコピーできるようにする
*   Maven の `maven-dependency-plugin` の `copy` 実行を追加する
    *   フェーズは `process-resources`（最小で扱いやすい）
    *   出力先は `src/main/liberty/config/resources/jdbc`
    *   既に同様の execution がある場合は **追記/再利用**し、重複作成しない

例（挿入イメージ）：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <executions>
    <execution>
      <id>copy-jdbc-driver-to-liberty</id>
      <phase>process-resources</phase>
      <goals>
        <goal>copy</goal>
      </goals>
      <configuration>
        <artifactItems>
          <artifactItem>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <!-- version は dependencyManagement なら省略も可。省略できない場合は ${jdbc.driver.version} -->
            <version>${jdbc.driver.version}</version>
            <outputDirectory>${project.basedir}/src/main/liberty/config/resources/jdbc</outputDirectory>
          </artifactItem>
        </artifactItems>
      </configuration>
    </execution>
  </executions>
</plugin>
```

> 既に `src/main/liberty/config/resources/jdbc` が無ければ作る。

## Step 4) `server.xml` を更新（library → jdbcDriver → dataSource）

### 4-1) `<featureManager>` に JDBC feature を入れる（無ければ）

*   既に `jdbc-*` があれば変更しない
*   無ければ Java バージョンを推定して追加：
    *   `maven-compiler-plugin` / `maven.compiler.release` / `maven.compiler.target` を見て
    *   11 以上なら `jdbc-4.3`、それ以外は `jdbc-4.2`
*   `<featureManager>` が無ければ作成するが、既存の順序/コメントは維持

### 4-2) `resources/jdbc` を参照する `<library>` を追加

*   追加する `id` は衝突しないように `jdbcLib-{dbType}` を基本にする
*   既に同じ id があれば再利用し、fileset だけ整合させる

例：

```xml
<library id="jdbcLib-postgres">
  <fileset dir="${server.config.dir}/resources/jdbc" includes="*.jar"/>
</library>
```

### 4-3) `<jdbcDriver>` を追加

*   `id` は `jdbcDriver-{dbType}`
*   `libraryRef` は上で作った library を参照

例：

```xml
<jdbcDriver id="jdbcDriver-postgres" libraryRef="jdbcLib-postgres"/>
```

### 4-4) 認証情報（`<authData>`）と `<dataSource>` を追加

*   `authData` は `id="dbAuth-{dbType}-{dbName}"` を基本（衝突回避）
*   password が `env:` 指定なら `${env.VAR}` を使用
*   `dataSource` は以下を基本：
    *   `id="ds-{dbType}-{dbName}"`
    *   `jndiName="jdbc/{dbName}"`
    *   `jdbcDriverRef="jdbcDriver-{dbType}"`
    *   `containerAuthDataRef="dbAuth-..."`

データソースのプロパティは URL 方式（最小）で追加する：

*   `jdbcUrl` を `properties` に `url="..."` もしくは vendor properties に適用できる形で入れる  
    （Liberty 構成の揺れを避けるため、**最小は URL を優先**）

例（最小形：URL 指定）：

```xml
<authData id="dbAuth-postgres-mydb" user="myuser" password="${env.DB_PASSWORD}"/>

<dataSource id="ds-postgres-mydb" jndiName="jdbc/mydb" jdbcDriverRef="jdbcDriver-postgres"
            containerAuthDataRef="dbAuth-postgres-mydb">
  <properties url="jdbc:postgresql://db.example.com:5432/mydb"/>
</dataSource>
```

> 既に同名 `jndiName` の dataSource が存在したら **新規追加せず**、差分候補として「既存を更新するか」だけ最小限質問する。

## Step 5) 接続チェック（自動で“実行”）

目的：**Liberty 起動前に、ドライバーとネットワーク/認証が通るか**を最小コストで確認。

実施内容：

1.  `src/main/liberty/config/resources/jdbc` に JAR が存在するか確認（無ければ build 実行を案内）
2.  一時 Java コード（`JdbcPing.java`）を `target/` に生成し、以下を実行：
    *   コンパイル：`javac`
    *   実行：`java -cp "<driverJar>[:...]" JdbcPing <jdbcUrl> <user> <password>`
3.  成功/失敗を表示
    *   成功：接続成功と接続先（host:port/dbName）を表示（password は出さない）
    *   失敗：例外クラスと主要メッセージ、想定原因（DNS/疎通/認証/SSL/ドライバー不一致）を分類して提示

> `prompt` 入力の password はファイルにもログにも書かない。  
> `env:` 指定の場合は Java 実行時にも同じ環境変数を参照して接続する（平文引数にしない）。

***

# 失敗時の方針（中断しない）

*   `pom.xml` への追記が失敗 → 変更点を出しつつ、手動修正の最小案を提示
*   `server.xml` の編集箇所特定が失敗 → 候補の server.xml を列挙し、選択だけ質問
*   接続チェックが失敗 → エラーを分類して次アクションを提示（例：ポート開放、DB の認証方式、SSL 必要有無、URL 形式など）

***

# 出力フォーマット（固定）

1.  解析した入力（dbType/host/port/dbName/user、password は **マスク** or env 名）
2.  更新したファイル一覧（`pom.xml`, `server.xml`）と編集概要
3.  `pom.xml` 追加差分（該当ブロックのみ）
4.  `server.xml` 追加差分（該当ブロックのみ）
5.  接続チェック結果（✅/❌、原因分類、次アクション）
6.  追加質問（必要な場合のみ：server.xml 複数、既存 jndiName 衝突など）

***

# 最小の追加質問ルール

質問は以下のときだけ：

*   server.xml が複数見つかり対象が一意に決まらない
*   `jndiName="jdbc/{dbName}"` が既に存在し、上書きが必要か判断できない
*   `prompt` 指定のパスワード入力（これ自体は質問というより入力要求）

***

## 付記（推奨）

*   password は可能なら `env:DB_PASSWORD` を使う（Git への平文混入を回避）
*   既存の Liberty 設計（`configDropins`、`server.env`、`bootstrap.properties`）がある場合は尊重し、同じ流儀に寄せる

***

## ここから実行例（使い方）

*   PostgreSQL（環境変数で password）

<!---->

    /liberty-datasource-create postgres db.example.com:5432 mydb myuser env:DB_PASSWORD

*   SQL Server（port 省略 → 1433）

<!---->

    /liberty-datasource-create mssql sql.example.com mydb sa prompt
