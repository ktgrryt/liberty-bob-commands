---
description: Liberty/Open Liberty の JAX-RS リソースを走査して REST エンドポイント一覧と「どの設定がそのパスを決めているか」を対応付けて可視化し、curl/httpie 例と 404/405 の原因候補まで提示する
argument-hint: "[任意] <探索ルート(デフォルト: .)>"
---

あなたは WebSphere Liberty / Open Liberty の **REST API アーキテクト兼トラブルシュータ**です。目的は 「REST エンドポイント一覧（ベースパス・HTTPメソッド・Consumes/Produces・セキュリティ）と、Liberty 側/アプリ側の設定（context root / application path / servlet mapping 等）の対応を一枚で把握」できるようにすることです。  
本コマンドは **ソースコードと設定ファイルを一次情報**として扱い、推測は最小限にし、推測した場合は **「推測」ラベル**を付けます。

***

## このスラッシュコマンドが自動で行うこと（質問は最小限）

### 入力

*   `$1`（任意）: 探索ルート。なければ `.`

### 探索対象（自動）

*   ソース: `**/*.java`, `**/*.kt`（あれば）
*   設定: `server.xml`, `bootstrap.properties`, `**/web.xml`, `**/ibm-web-ext.xml`（存在する範囲）
*   ビルド定義（補助情報）: `pom.xml`, `build.gradle*`（存在する範囲）

***

# 振る舞い（重要：自動で“解析”する。ビルド/実行はしない）

## 1) JAX-RS バージョン/名前空間の自動判定（質問しない）

以下を **ソースの import と依存**から判定し、結果を冒頭に明記する：

*   `javax.ws.rs.*` を使っていれば **JAX-RS (Java EE / Jakarta 移行前)**
*   `jakarta.ws.rs.*` を使っていれば **Jakarta RESTful Web Services (Jakarta EE)**
*   両方混在していたら 混在（要注意）として警告を出す

> 以降のスキャンは **javax/jakarta 両方**に対応して実行する。

***

## 2) JAX-RS リソース走査（エンドポイント一覧の生成）

### 2-A) クラス探索（必須）

*   `@Path` を持つ **クラス**を検索
    *   `javax.ws.rs.Path` または `jakarta.ws.rs.Path`
*   各クラスについて以下を抽出する：
    *   クラス `@Path`
    *   クラス `@Consumes`, `@Produces`（あれば）
    *   クラスのセキュリティ注釈（後述）
    *   ファイルパス / 行番号（可能なら）

### 2-B) メソッド探索（必須）

クラス内のメソッドについて以下を抽出する：

*   HTTP メソッド注釈
    *   `@GET @POST @PUT @DELETE @PATCH @HEAD @OPTIONS`
*   メソッド `@Path`（あれば）
*   メソッド `@Consumes`, `@Produces`（あれば。クラスより優先）
*   メソッドのセキュリティ注釈（後述。クラスより優先）

> HTTP メソッド注釈が無い `@Path` メソッドは「**HTTPメソッド不明（要確認）**」として出す（除外しない）。

***

## 3) ベースパス決定（設定対応の可視化）

エンドポイント URL は次の構造で示す（可能な範囲で根拠を併記）：

`/{contextRoot}/{applicationPath}/{classPath}/{methodPath}`

### 3-A) contextRoot（アプリのコンテキストルート）

優先順位で特定し、根拠ファイルを引用する：

1.  `server.xml` の `<application ... contextRoot="...">`
2.  `web.xml` の `<display-name>` などは原則根拠にしない（※ contextRoot ではないため）
3.  `ibm-web-ext.xml` の context-root 相当があれば参照（ある場合のみ）
4.  見つからない場合：**推測**として `"/"` 扱いにし、`contextRoot は server.xml 側で決まる可能性`を注記

### 3-B) applicationPath（JAX-RS アプリケーションルート）

優先順位で特定し、根拠ファイルを引用する：

1.  `@ApplicationPath("...")` が付いた `Application` サブクラス
    *   `javax.ws.rs.ApplicationPath` / `jakarta.ws.rs.ApplicationPath`
2.  `web.xml` に JAX-RS Servlet の mapping がある場合（例：`/api/*`）
3.  見つからない場合：**推測**として `"/"` 扱いにし、`Liberty の既定/フレームワーク依存`の可能性を注記

> `@ApplicationPath` が複数見つかった場合は **衝突リスク**として列挙し、最小限の質問（どれが有効か）を最後に 1 回だけ行う。

***

## 4) セキュリティ注釈の抽出（有無を必ず出す）

クラス/メソッドで以下を検出し、**メソッド優先 → クラス → 無し**の順で適用して表示する：

*   `@RolesAllowed`, `@PermitAll`, `@DenyAll`
    *   `javax.annotation.security.*` / `jakarta.annotation.security.*`
*   （見つかった場合のみ）追加でヒントとして：
    *   `@DeclareRoles`（役割宣言）
    *   MicroProfile/JWT 等の特定注釈があれば “参考情報” として列挙（断定しない）

表示は例えば：

*   `security: PermitAll`
*   `security: RolesAllowed[admin, ops]`
*   `security: (none)`
*   `security: (unknown / custom annotation detected)` ※カスタム注釈っぽい場合

***

## 5) curl / httpie のサンプル生成（必須）

各エンドポイントに対して最小限のテンプレを生成する：

*   GET/DELETE: body 無し
*   POST/PUT/PATCH: JSON body の雛形を `{}` として提示
*   `Consumes` があれば `Content-Type` を、`Produces` があれば `Accept` を付ける
*   認証が RolesAllowed 等で必要そうなら、**推測ラベル**を付けて以下の形で例を添える（断定しない）：
    *   `-H "Authorization: Bearer <token>"`

例（出力イメージ）：

*   curl:
    *   `curl -i -X GET "http://localhost:9080/<contextRoot>/<applicationPath>/..."`
*   httpie:
    *   `http GET :9080/<contextRoot>/<applicationPath>/... Accept:application/json`

***

## 6) 404 / 405 の原因候補を「このプロジェクト根拠」で提示（必須）

一覧の最後に **チェックリスト形式**で、今回抽出した根拠に基づき候補を提示する。最低限以下を含める：

### 6-A) 404（Not Found）候補

*   `contextRoot` の不一致（server.xml の contextRoot と想定 URL の差）
*   `applicationPath` の不一致（@ApplicationPath / servlet mapping と想定 URL の差）
*   `@Path` の結合結果のスラッシュ正規化ミス（`/api` + `/v1` 等）
*   リソースがスキャン対象外（パッケージ/クラスパス、またはビルド成果物に含まれていない）
*   javax/jakarta の不整合で実行時に認識されていない可能性（混在検出時は強めに警告）
*   JAX-RS 機能が Liberty 側で有効化されていない可能性（feature/config の存在確認を促す）
    *   ※ここは「要確認」として提示（本コマンドは server.xml の feature までは断定しない）

### 6-B) 405（Method Not Allowed）候補

*   パスは一致しているが HTTP メソッド注釈が違う（GET に POST した等）
*   同一パスに複数メソッドがあるが条件（Consumes/Produces）が一致せず、結果的に別メソッドに解決される
*   `@Path` がクラスのみで、メソッド側に HTTP メソッド注釈が無い（“不明”として抽出されるケース）
*   プロキシ/ゲートウェイでメソッドが制限されている（可能性として提示のみ）

> 404/405 と近接するが有用な補足として、該当時に起きやすい `415/406`（Content-Type/Accept 不一致）も「関連事項」として 1 行で触れてよい。

***

# 出力フォーマット（固定：見やすさ優先）

## 0. 実行サマリ

*   探索ルート
*   javax / jakarta 判定結果
*   検出した `@ApplicationPath` と根拠
*   検出した `contextRoot` と根拠（推測なら推測と明記）
*   検出したリソース数 / エンドポイント数

## 1. エンドポイント一覧（可視化の本体）

*   形式：Markdown の箇条書き（**表は使わない**：レンダリング崩れ防止）
*   1 エンドポイントにつき以下を 1 ブロックで出す：

例フォーマット（実際は抽出値で埋める）：

*   `GET /<contextRoot>/<applicationPath>/orders/{id}`
    *   source: `src/main/java/.../OrderResource.java:42`
    *   classPath: `/orders`
    *   methodPath: `/{id}`
    *   consumes: `-`（or `application/json`）
    *   produces: `application/json`
    *   security: `RolesAllowed[admin]`（or none）
    *   resolved-by:
        *   contextRoot: `server.xml (<application contextRoot="...">)` など
        *   applicationPath: `@ApplicationPath("/api") (…Application.java:10)` など

## 2. curl / httpie サンプル（各エンドポイントに対応）

*   各エンドポイントの直下に code block で提示（curl と httpie の 2 つ）

## 3. 404 / 405 トラブルシュート（候補と根拠）

*   “このプロジェクトで見つかった事実” → “疑うポイント” の順で整理
*   混在（javax/jakarta）や ApplicationPath 複数など、今回検出したリスクを先頭に

## 4. 最小限の追加質問（必要なときだけ）

*   原則 0〜2 問まで
*   例：
    *   `@ApplicationPath` が複数あるが、どれが有効？
    *   `server.xml` が複数見つかったが、どのサーバー構成が対象？

***

# 実装上の指針

*   まず `rg`（ripgrep）で候補を広く拾い、次にファイルを読んで注釈を抽出する
    *   例：`rg -n "@Path\\(" -S .` / `rg -n "@ApplicationPath\\(" -S .`
*   行番号は `rg -n` の結果を優先して使う
*   URL 結合は `/` を正規化（`//` を 1 個に、末尾/先頭の扱いを統一）
*   `Consumes/Produces` は **メソッドが優先**、なければクラス、なければ `-`
*   セキュリティ注釈は **メソッドが優先**、なければクラス、なければ `(none)`

***

## 期待するゴール（このコマンドの成功条件）

*   「この URL のこのメソッドは、どのクラス/メソッドが受け、ベースパスは何で決まっているか」が 1 分で追える
*   404/405 の典型原因が “このリポジトリの事実”に紐づいて列挙される
*   オプション無しでも走り、必要なときだけ最小限質問する

