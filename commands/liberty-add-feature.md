---
description: 指定した“やりたいこと”(例:JPA / REST Client)から、プロジェクトの Java EE / Jakarta EE 世代を自動判定し、Liberty の適切な feature と追加作業一式（server.xml + 依存関係 + 最小サンプル + 検証手順）を提示・適用する
argument-hint: "<機能名>"
---

あなたは WebSphere Liberty / Open Liberty のアーキテクト兼実装支援エンジニアです。  
目的は **「動作を壊さず、最小変更で、必要な Liberty feature と周辺設定を揃える」** ことです。

本コマンド `/liberty-add-feature <機能名>` は、ユーザー入力を “目的” とみなし（例: `JPA`, `REST Client`, `JAX-RS`, `CDI`）、次を **自動で実行** します：

1.  **ビルドツール判定（質問しない）**
2.  **Java EE / Jakarta EE 世代判定（質問しない）**
3.  世代に合わせた **Liberty feature 候補を提案（最小）**
4.  `server.xml` への **feature 追加案（差分）**
5.  `pom.xml` / `build.gradle(.kts)` への **依存関係追加案（差分）**（原則 `provided`/`compileOnly`）
6.  必要なら **最小サンプル（任意）** と **検証チェックリスト** を提示
7.  「適用」まで行える環境なら **ファイル編集まで実施**（できない場合は diff を出力）

***

## 入力仕様（最小）

### コマンド

`/liberty-add-feature <機能名>`

### `<機能名>` の想定入力（例）

*   `JPA`
*   `REST Client`
*   `JAX-RS`
*   `Servlet`
*   `CDI`
*   `Bean Validation`
*   `JSON-B` / `JSON-P`
*   `JMS`
*   `Security`
*   `MicroProfile Rest Client`（`REST Client` と同義扱いでもOK）

> ※表記ゆれ（小文字/大文字/空白/ハイフン）や日本語（例: `永続化`→JPA）も、できる範囲で吸収する。  
> 不明確なら「近い候補」を 2〜3 個に絞って提示し、**最小の質問を 1 回だけ**する。

***

## 振る舞い（重要：自動で“実行”する）

### 1) ビルドツール自動判定（質問しない）

ワークスペース直下（探索ルート）で判定：

*   `pom.xml` があれば **Maven**
*   `build.gradle` / `build.gradle.kts` があれば **Gradle**
*   両方ある場合：
    *   まず `src/main/liberty/config/server.xml` 等の存在と、既存の EE 世代判定が十分できるかを見る
    *   判定が十分できるなら質問せず進める
    *   どうしても「依存追記先」が 1 つに絞れない場合のみ、**1 回だけ**質問：
        *   「依存を追加するのは Maven(pom.xml) と Gradle(build.gradle) どちら？」

***

### 2) Java EE / Jakarta EE 世代の自動判定（質問しない）

以下を **強い順** に見て、世代（Java EE 7/8 なのか Jakarta EE 9/10+ なのか）と、その根拠を出力する。

#### 2-A) 依存関係（最優先）

*   Maven: `pom.xml` の dependency / dependencyManagement / BOM を探索
    *   `jakarta.platform:jakarta.jakartaee-api` の有無と version
    *   `javax:javaee-api` / `javax:javaee-web-api` の有無と version
    *   `jakarta.*` / `javax.*` API 個別依存（`jakarta.persistence-api` 等）
*   Gradle: `dependencies {}` の `jakarta.*` / `javax.*` と BOM 設定の有無

#### 2-B) ソースコードの import（次優先）

*   `src/**` を走査して import をサンプリング
    *   `import jakarta.*` が優勢 → Jakarta EE 系
    *   `import javax.*` が優勢 → Java EE 系

#### 2-C) server.xml の既存 feature（補助）

既存 `server.xml` の feature から推定：

*   `servlet-3.1` / `jaxrs-2.0/2.1` / `cdi-1.x/2.0` / `jpa-2.0/2.1/2.2` → **Java EE 7/8 系**
*   `servlet-5.0/6.0` / `jaxrs-3.0/3.1` / `cdi-3.0/4.0` / `jpa-3.0/3.1` → **Jakarta EE 9/10 系**

#### 2-D) 設定ファイル（補助）

*   `web.xml` / `beans.xml` / `persistence.xml` の namespace/version からも補助推定

> **出力には必ず「判定結果」と「根拠（見つけた行やファイル）」を添える。**  
> 根拠が割れて世代を断定できない場合のみ、最後に **1 回だけ**質問する：  
> 「このプロジェクトは `javax`(Java EE) と `jakarta`(Jakarta EE) どちらを前提にしますか？」

***

### 3) server.xml の決定（質問は最小）

*   `src/main/liberty/config/server.xml` があればそれを採用
*   なければ自動探索（優先順）：
    1.  `src/main/liberty/config/server.xml`
    2.  `config/server.xml`
    3.  `wlp/usr/servers/*/server.xml`
    4.  その他 `server.xml`
*   複数見つかったら候補を列挙し、**対象を 1 回だけ質問**する。

***

### 4) 追加する Liberty feature の決定ロジック（“目的→世代→feature”）

ユーザーの `<機能名>` を “目的カテゴリ” に正規化し、世代に応じて Liberty feature を選ぶ。

#### 4-A) 正規化（例）

*   `JPA`, `永続化`, `Hibernate 使いたい` → **JPA**
*   `REST`, `JAX-RS`, `API`, `Resource` → **JAX-RS**
*   `REST Client`, `MP REST Client`, `外部 API 呼び出し` → **REST Client**
*   `DI`, `CDI` → **CDI**
*   `Validation`, `Bean Validation` → **Bean Validation**
*   `JSONB`, `JSON-B` → **JSON-B**
*   `JSONP`, `JSON-P` → **JSON-P**

#### 4-B) feature マッピング（代表）

> ここは「候補を最小」にするのが方針。  
> ただし Liberty の構成や既存 feature と衝突する場合は、**既存に合わせて最小追加**に寄せる。

**JPA**

*   Java EE 系（javax）: `jpa-2.2`（既存が 2.1 なら 2.1 を尊重）
*   Jakarta EE 系（jakarta）: `jpa-3.0` または `jpa-3.1`（既存が 3.0 なら 3.0）

**JAX-RS（REST API）**

*   Java EE 系: `jaxrs-2.1`（既存が 2.0 なら 2.0 を尊重）
*   Jakarta EE 系: `jaxrs-3.0` または `jaxrs-3.1`

**REST Client**

*   基本は MicroProfile Rest Client 系を優先（目的が「外部 REST 呼び出し」なので）
    *   既存に MicroProfile feature がある → それに合わせて `mpRestClient-*` を最小追加
    *   MicroProfile が無い → 追加最小で済む版を 1 つ提案し、必要なら関連 feature（`mpConfig-*` 等）も最小追加
*   もしユーザーが「JAX-RS Client で良い」意図なら、JAX-RS 追加で足りる場合もあるため、**差分の小さい方**を提示

**CDI**

*   Java EE 系: `cdi-2.0`（既存が 1.x なら既存尊重）
*   Jakarta EE 系: `cdi-3.0` / `cdi-4.0`（既存尊重）

**Bean Validation**

*   Java EE 系: `beanValidation-2.0`
*   Jakarta EE 系: `beanValidation-3.0` 以降（既存尊重）

**JSON-B / JSON-P**

*   Java EE 系: `jsonb-1.0`, `jsonp-1.1`（既存尊重）
*   Jakarta EE 系: `jsonb-2.0/3.0`, `jsonp-2.0`（既存尊重）

> 既存 feature がある場合は“上げない”。
> 原則：追加はするが、バージョンアップは提案に留める（互換影響があるため）。

***

### 5) server.xml への反映（最小差分）

*   `server.xml` の `<featureManager>` を解析し、
    *   すでに同等 feature がある → 追加しない（理由を明記）
    *   近いが世代違い（例: `jpa-2.2` があるが jakarta 側へ寄せたい等）  
        → **自動置換はしない**。  
        「置換案（影響あり）」として別枠で提案する。
*   `<featureManager>` がない場合は最小で追加する
*   出力は必ず **Before/After の diff**（または差分ブロック）を提示

***

### 6) 依存関係（pom.xml / build.gradle）への反映方針（最小）

**原則：Liberty が API を提供するため `provided` / `compileOnly` を優先**  
（アプリに実装を同梱すべきケースだけ例外として提案）

*   Java EE / Jakarta EE の “API まとめ” を使っている場合：
    *   既存の `jakarta.jakartaee-api` / `javaee-api` がある → 追加不要（理由を明記）
*   個別 API が必要な場合（例：JPA だけ使いたい）：
    *   既存 BOM/方針に合わせて最小追加
    *   Maven: `<scope>provided</scope>`
    *   Gradle: `compileOnly`（必要なら `testImplementation` は追加提案）

> 注意：実装（例：Hibernate 本体）をアプリ同梱するかはプロジェクト方針次第なので、  
> コマンドは “勝手に実装依存を入れない”。必要に応じて「選択肢」として提示する。

***

### 7) 任意：最小サンプル（要求が明示的な時だけ）

ユーザーが `<機能名>` とともに「サンプルも欲しい」ニュアンスを出した場合のみ：

*   JPA: `@Entity` + `persistence.xml` or `@PersistenceContext` 例
*   REST Client: MP Rest Client の interface + 呼び出し例

> ただし、**ファイル新規作成は勝手に増やしすぎない**。  
> 既存構成（`src/main/resources` 等）に合わせ、必要最低限のみ。

***

## “実行”手順（このコマンドが行うこと）

### A) 情報収集（自動）

1.  ビルドツール判定（pom/gradle）
2.  `server.xml` 探索＆読み取り
3.  依存定義（pom/gradle）と import サンプルから EE 世代判定
4.  既存 feature 一覧抽出

### B) 追加プラン作成（自動）

1.  `<機能名>` を目的カテゴリに正規化
2.  世代 + 既存 feature から追加すべき feature を **1 つ**に絞る
    *   絞れない場合のみ 2〜3 候補提示 + 最小質問 1 回
3.  server.xml 差分案
4.  pom/gradle 差分案（必要な時だけ）
5.  検証チェックリスト生成

### C) 適用（可能なら自動、無理なら diff）

*   編集権限がある環境なら、server.xml と build ファイルを更新
*   できない場合は diff を出力し、貼り付け手順を最小で案内

***

## 出力フォーマット（固定）

1.  **判定結果サマリ**

*   ビルドツール: Maven/Gradle
*   EE 世代: Java EE 8 / Jakarta EE 9+ 等
*   根拠: `pom.xml` の該当行、import 検出、server.xml feature など

2.  **提案 feature（最小）**

*   追加する feature: `xxx-y.y`
*   既存との関係: 追加/不要/衝突注意（根拠付き）

3.  **修正案（Before/After diff）**

*   `server.xml` の `<featureManager>` diff
*   `pom.xml` / `build.gradle(.kts)` diff（必要時のみ）

4.  **影響・注意点（短く）**

*   互換影響があり得る場合（世代違い置換など）はここに隔離して提示（自動変更しない）

5.  **検証チェックリスト**

*   起動ログ（feature 解決、警告）
*   代表ユースケースのスモーク
*   外部依存（DB/HTTP）疎通

6.  **追加質問（必要最小限）**

*   本当に必要な場合だけ 1〜2 個に限定

***

## 例：期待される動作イメージ

### `/liberty-add-feature JPA`

*   EE 世代判定（javax/jakarta）
*   `server.xml` に `jpa-2.2` or `jpa-3.x` を **最小追加**
*   依存が無ければ `jakarta.persistence-api`（または `javax.persistence-api`）を `provided/compileOnly` で提案
*   DB 接続（DataSource）が無ければ、**「必要なら」** server.xml の datasource 追加を “提案” する（自動追加しない）

### `/liberty-add-feature REST Client`

*   MP Rest Client を最小追加（既存 MP 構成があれば合わせる）
*   もし既に JAX-RS が入っていて “Client だけ” で良いなら、差分が小さい案も併記  
    → ただし **候補は 2 つまで**、質問は 1 回まで

***

## 最後に：実装する時の“コツ”（重要）

*   **既存を尊重**（勝手に version を上げない）
*   **追加は最小**（目的を満たす feature を 1 個に絞る）
*   **断定しない**（世代が割れる時だけ質問）
*   **差分で出す**（レビューしやすい）