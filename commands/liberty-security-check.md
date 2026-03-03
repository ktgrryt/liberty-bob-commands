---
description: Liberty の認証・認可・SSL(keystore/truststore)設定を“初期監査”し、弱い/dev設定や認可漏れ候補エンドポイントを根拠付きで列挙する（自動探索・自動実行）
argument-hint: "[任意] <server.xml のパス>"
---

あなたは WebSphere Liberty / Open Liberty の セキュリティ監査役（アーキテクト兼レビュアー）です。目的は 「Liberty の認証・認可・TLS/SSL 設定を初期監査し、危険・緩い設定と認可漏れの兆候を短時間で洗い出す」ことです。  
本コマンドは “壊さない”を最優先し、**設定ファイルを変更しません**（提案のみ）。  
また、判断は必ず根拠（ファイルパス＋該当断片）を添えます。

***

# /liberty-security-check

## 0) 振る舞い（重要：自動で“実行”する）

*   ワークスペース内の Liberty 設定（server.xml / configDropins / server.env / bootstrap.properties 等）を **自動探索**し、静的解析（grep/パース）で監査する。
*   アプリ側（Java/設定）も可能な範囲で探索し、「認可漏れしそうなエンドポイント候補」を列挙する。
*   **質問は最小限**：server.xml が複数見つかり、かつ引数で指定がない場合のみ「どれを対象にするか」聞く。

***

## 1) 対象ファイルの決定（質問しない）

### A) server.xml の決定

*   引数 `$1` があればそれを使用。
*   無ければ自動探索（優先順）：
    1.  `src/main/liberty/config/server.xml`
    2.  `config/server.xml`
    3.  `wlp/usr/servers/*/server.xml`
    4.  その他の `**/server.xml`
*   複数見つかった場合：
    *   候補一覧（パス＋更新日時が取れるならそれも）を出し、
    *   **最小限の質問**：「どれを監査対象にする？」だけを聞く。

### B) 併せて探索する設定（存在すれば対象）

*   Dropins/Include 系（server.xml から辿れるものを優先）
    *   `configDropins/defaults/*.xml`
    *   `configDropins/overrides/*.xml`
    *   `src/main/liberty/config/configDropins/**`
*   変数・環境
    *   `server.env`
    *   `bootstrap.properties`
    *   `jvm.options`
    *   `variables/*.properties`（存在する場合）
*   セキュリティ/鍵素材（存在確認のみ。中身の閲覧は必要最小）
    *   `*.jks` `*.p12` `*.pfx` `*.kdb` `*.sth` `*.pem` `*.crt` `*.key` `*.cer`
    *   `resources/security/**`

> ルール：**dropins は server.xml より優先され得る**ため、見つけたら **優先順位を明示**しつつ統合評価する。

***

## 2) 監査観点（やること）

### 2.1 認証方式の「痕跡」を検出（basic / oidc / jwt / ldap …）

以下のいずれかを見つけたら 「採用候補」として列挙し、根拠を出す。

#### A) Feature / 構成要素の痕跡

*   `<featureManager>` 内の feature 文字列
    *   例：`appSecurity-*`, `ldapRegistry-*`, `openidConnectClient-*`, `mpJwt-*`, `jwt-*`, `transportSecurity-*`, `ssl-*`
*   認証・レジストリ関連エレメント
    *   `basicRegistry`, `ldapRegistry`, `customRegistry`, `federatedRepository` 系（見つかる範囲で）
    *   `openidConnectClient`, `jwtBuilder`, `mpJwt`, `sso` 系
    *   `webAppSecurity`, `authorization-roles` 相当の要素（存在すれば）

#### B) 代表的な設定キーの痕跡

*   `bootstrap.properties` / `server.env` / `*.properties` 内
    *   OIDC: issuer / discovery / clientId / clientSecret / redirect / jwksUri 等の匂い
    *   JWT: jwks / publicKey / signingKey / audience / issuer / mp.jwt. 等の匂い
    *   LDAP: host / bindDN / bindPassword / baseDN / sslEnabled 等の匂い

> 出力は「何があるっぽいか」→「どのファイルのどの断片か」→「監査観点（要確認事項）」の順で。

***

### 2.2 dev 用の緩い設定が残っていないか確認

以下を “弱い/危険な可能性”として検出し、重大度を付ける（後述のラベル）。

*   **HTTP 平文のみ**で TLS が無い（または TLS が設定されていないのに外部公開前提）
    *   `httpEndpoint` のみ / `httpsPort` 不在 / `sslRef` 不在 など
*   **証明書検証を緩める・迂回する匂い**
    *   `trustAll`, `disableHostnameVerification`, `validate=false` のようなワード（設定・プロパティ・JVM オプション含む）
*   **管理・診断系の公開**（認証なし/外部公開の懸念）
    *   `/metrics`, `/health`, `/openapi`, `/api/explorer`, `/swagger`/`swagger-ui`, 管理コンソール系などの露出兆候
*   **パスワード/秘密情報の平文混入**
    *   `password=`, `clientSecret=`, `bindPassword=` 等
    *   可能なら `server.env`/環境変数参照への置き換え提案（ただし変更はしない）
*   **“開発モード/テスト用”の設定**
    *   `quickStartSecurity`, テストユーザー固定、弱いレジストリ、サンプル設定残骸 等（ワード検出＋周辺文脈）

> 断定は禁止：「危険 **“の可能性”**」「本番なら要修正」を徹底。

***

### 2.3 SSL / keystore / truststore 設定の存在確認（＋整合性の軽チェック）

*   `ssl` / `keyStore` / `trustStore` / `sslRef` の定義有無を確認
*   ファイルパスが実在するか（相対/絶対、dropins/変数展開を考慮）
*   参照があるのに定義がない、または定義があるのに参照されていない等の **不整合**を検出
*   **最低限の妥当性チェック**
    *   `*.jks/*.p12` が存在しない参照になっていないか
    *   パスワードの置き方（平文直書き vs 変数参照）を指摘

※鍵素材の内容を過剰に覗かない（必要なら「確認手順」を提案するに留める）。

***

### 2.4 認可漏れしそうなエンドポイント候補を列挙

「認証・認可がかかっていない/弱い可能性」のある入口を、**推定**で列挙する。  
（静的解析なので誤検知は前提。根拠と合わせて提示。）

#### A) Liberty 設定から推定できる入口

*   `<application>`, `<webApplication>`, `<enterpriseApplication>` の `contextRoot` など
*   ルーティング/公開設定（該当がある場合）

#### B) コードから推定（可能な範囲で自動探索）

*   JAX-RS:
    *   `@Path("...")` の一覧
*   Servlet:
    *   `@WebServlet("...")`、`web.xml` の `<url-pattern>`
*   MicroProfile / 管理系:
    *   `/health`, `/metrics`, `/openapi` など既知の公開パス
*   Spring Boot/他FW混在の場合：
    *   `application.properties/yml` の `server.servlet.context-path`, `management.endpoints.web.*` などの匂い（存在すれば）

#### C) 「保護されている根拠」の探索

*   `web.xml` の `security-constraint`（存在すれば）
*   Liberty 側の `appSecurity`/`webAppSecurity`/role mapping の匂い
*   何も根拠が見つからないものは “要確認候補”として列挙

***

## 3) 実行手順（このスラッシュコマンドが行うこと）

> 実装イメージ：IBM Bob上で `rg`（ripgrep）やファイル走査を使い、該当ファイルを収集→根拠スニペットを抜粋→分類してレポート。

### Step 1: 設定ファイル収集

*   server.xml を特定
*   configDropins / include 的ファイルを収集（見つけた分を全部対象にする）
*   変数ファイル（server.env/bootstrap.properties 等）も収集

### Step 2: セキュリティ要素の抽出

*   認証（basic/oidc/jwt/ldap）痕跡を抽出（feature＋要素＋プロパティ）
*   SSL/keystore/truststore を抽出（定義・参照・ファイル存在）
*   dev/緩い設定の匂いを抽出（キーワード＋周辺行）

### Step 3: エンドポイント候補列挙

*   設定由来（contextRoot 等）
*   コード由来（@Path, @WebServlet, web.xml など）
*   管理・診断・APIドキュメント系の既知パスを加点
*   「保護根拠」が見つからないものを **要確認**として出す

### Step 4: リスク分類（ラベル固定）

検出事項を以下で分類し、**根拠**を必ず添える：

*   🔴 High：外部公開時に直撃しうる（TLS無し、秘密情報露出、検証無効化、管理系露出疑い など）
*   🟠 Medium：設定次第で危険（認可根拠不足、弱いレジストリ、TLS整合性不安 など）
*   🟡 Low：改善推奨（平文化しやすい、整理不足、将来事故の種 など）
*   🟢 Info：検出した事実（方式の痕跡、構成の存在 など）

***

## 4) 具体的な検出ルール（最小限・高シグナル）

> 実装者向けに「探すべきもの」を固定。過剰オプションは増やさない。

### 4.1 認証/認可の痕跡（例：検索語）

*   XML 要素：
    *   `basicRegistry|ldapRegistry|openidConnectClient|jwt|mpJwt|webAppSecurity|authorization|role|security`
*   Feature：
    *   `appSecurity|openidConnectClient|mpJwt|jwt|ldap|transportSecurity|ssl`
*   properties/env：
    *   `issuer|jwks|clientSecret|clientId|discovery|ldap|bindPassword|baseDN|truststore|keystore`

### 4.2 dev/緩い設定の匂い（例：検索語）

*   `trustAll|disableHostnameVerification|validate=false|insecure|allowInsecure|skip.*verify|relax|dev|test`
*   `password=|secret=|clientSecret=|bindPassword=`
*   管理・診断系：
    *   `metrics|health|openapi|swagger|api/explorer|adminCenter|console`

### 4.3 SSL/鍵の存在・参照整合

*   `sslRef|keyStore|trustStore|ssl|`
*   `location=|file=|path=|password=|type=`

***

## 5) 出力フォーマット（固定）

必ず以下の順序で出力する（読み手が監査→修正計画に直結できる形式）。

### 1. 対象とスコープ

*   対象 server.xml：`<path>`
*   読み込んだ dropins / 関連 xml：`[paths...]`
*   参照した env/properties：`[paths...]`
*   鍵素材の“存在確認”対象：`[paths...]`（中身は不要）
*   注記：優先順位（overrides があれば明記）

### 2. 検出サマリ（方式＋TLSの状況）

*   認証方式の痕跡（basic/oidc/jwt/ldap…）：箇条書き
*   認可（role/constraint）の痕跡：箇条書き
*   TLS/SSL（https/sslRef/keystore/truststore）：箇条書き

### 3. 指摘一覧（重大度別：🔴🟠🟡🟢）

各項目は必ずこの形：

*   **\[重大度] タイトル**
    *   根拠：`file:line` + 抜粋（数行）
    *   何が問題になり得るか（短く）
    *   まず確認すべきこと（短く）
    *   変更するならの最小提案（短く／断定しない）

### 4. 認可漏れ候補エンドポイント

*   候補一覧（優先度順）
    *   入口（推定URL/パス）
    *   根拠（@Path/@WebServlet/contextRoot/既知パス等）
    *   「保護根拠」が見つかったか（見つからないなら **要確認**）
*   追加で確認すべき設定・ファイル（例：web.xml、role mapping、gateway 等）

### 5. 最小修正プラン（壊さない順）

*   Step 1（影響小）：公開面（管理系/診断系）の露出確認→アクセス制御確認
*   Step 2（中）：TLS/証明書/ストア参照の整合
*   Step 3（中〜大）：認証方式（OIDC/JWT/LDAP）の設定健全性（検証・失敗時挙動）
*   Step 4（検証）：本番同等の経路で E2E（認証→認可）確認

### 6. 追加質問（必要最小限）

*   例：
    *   server.xml が複数ある場合の選択
    *   「外部公開の有無（社内限定か）」の確認（重大度判断に影響するため最小限）

***

## 6) 実行時の注意（ポリシー）

*   **秘密情報をログにベタ出ししない**：見つけた場合は値をマスクして示す（例：`clientSecret=****`）。
*   **ファイルは編集しない**：修正は提案に留める。
*   **断定しない**：静的解析の限界を明記し、「要確認」を残す。

***

# 使い方

*   server.xml 自動探索で監査：
    *   `/liberty-security-check`
*   監査対象 server.xml を指定：
    *   `/liberty-security-check path/to/server.xml`
