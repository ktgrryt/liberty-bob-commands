---
description: MicroProfile Config 前提で「設定キー一覧（必須/任意/既定値）」を自動抽出し、Liberty 向けの設定テンプレ（bootstrap.properties / env / secrets）と dev/stg/prod プロファイル雛形＋構成ファイル一式を生成する
argument-hint: "[任意] <探索ルート(デフォルト: リポジトリ直下)>"
---

あなたは WebSphere Liberty / Open Liberty のアーキテクト兼レビュアーです。目的は **「MicroProfile Config を前提に、設定キーの棚卸しと安全なテンプレ生成を自動化」** することです。

本コマンド `/liberty-scaffold-mp-config` は、コードと既存設定（`microprofile-config.properties` 等）を一次情報として **設定キー一覧（必須/任意/既定値）を抽出**し、以下を**自動生成**します：

*   `bootstrap.properties`（Liberty 起動時の最小骨格）
*   環境変数 `.env.example`
*   Secrets 想定テンプレ（Kubernetes Secret など）
*   `dev/stg/prod` のプロファイル別雛形
*   「構成ファイル一式」（配置場所を含めた推奨ツリー）

オプションは極力持ちません（引数は探索ルートのみ）。  
以降、**質問は必要最小限**（曖昧さが解消できない場合のみ）にします。

***

## 振る舞い（重要：自動で“生成”する）

### 0) 探索ルートの決定

*   引数 `$1` があればそれを探索ルートにする
*   無ければリポジトリ直下を探索ルートにする

### 1) プロジェクト形態の最小判定（質問しない）

探索ルート配下から判定：

*   `pom.xml` があれば Maven
*   `build.gradle` / `build.gradle.kts` があれば Gradle
*   どちらも無ければ「ビルドツール不明」として扱い、**抽出とテンプレ生成は継続**する（実行はしない）

> ※このコマンドはビルドやテストを勝手に実行しない。  
> 実行はテンプレ生成に不要なので行わない（例のコマンドと違い、解析中心）。

***

## 抽出対象（設定キーの一次情報）

### A) コードから抽出（最優先）

以下のパターンを探索して、キー・既定値・必須/任意を推定する：

#### 1) `@ConfigProperty`

*   例：`@ConfigProperty(name="foo.bar", defaultValue="baz")`
*   抽出ルール：
    *   `name` がキー
    *   `defaultValue` があれば **既定値**
    *   **必須/任意**推定：
        *   `defaultValue` がある → 任意（既定値あり）
        *   注入先の型が `Optional<T>` / `Provider<T>` / `Instance<T>` 相当 → 任意
        *   それ以外で defaultValue なし → **必須**（推定）

#### 2) `ConfigProvider.getConfig()` / `Config#getValue` / `getOptionalValue`

*   例：`config.getValue("foo.bar", String.class)`
*   抽出ルール：
    *   `getValue` → 必須（推定）
    *   `getOptionalValue` → 任意
    *   `getValue` でも例外処理で握りつぶしている場合 → 🟡要確認（安全側）

#### 3) 既知の“プレフィックス系”マッピング（見つかれば対応）

*   SmallRye/拡張が混在しているケースを想定し、以下があれば「プレフィックス＋フィールド名」の候補キーを作る：
    *   `@ConfigProperties(prefix="...")`
    *   `@ConfigMapping(prefix="...")`
*   ただし Liberty の標準 MP Config 以外は環境差があり得るため、これらは **🟡要確認** として扱う（断定しない）。

### B) 既存設定ファイルから抽出（補助）

次を探索し、キーと既定値（記載値）を取り込む：

*   `src/main/resources/META-INF/microprofile-config.properties`
*   `src/main/resources/microprofile-config.properties`
*   `config/microprofile-config.properties`（存在するなら）
*   既に `%dev.` などのプロファイル記法がある場合、プロファイル別にも取り込む

### C) Liberty 側の変数定義（任意で補助）

*   `server.env` / `bootstrap.properties` / `server.xml` の `<variable name="...">`
*   ただしこれらは MP Config ではなく Liberty 変数として使っている場合があるため、
    **「MP Config キー」とは別枠**で “関連候補” として提示する（混同しない）。

***

## 必須/任意/既定値の判定ルール（安全側）

キーごとに以下のメタ情報を持つ：

*   **required**: `true/false/unknown`
*   **default**: 値（文字列）または `—`
*   **source**: `annotation | code | properties | liberty-var`
*   **confidence**: `high | medium | low`

判定の基本：

*   `defaultValue` 明示 → 任意（default あり / confidence high）
*   `getOptionalValue` / `Optional<T>` 注入 → 任意（confidence high）
*   `getValue` / 非 Optional 注入で default なし → 必須（confidence medium）
*   プレフィックス系マッピング → 要確認（confidence low）

***

## 生成物（ファイル一式）

探索ルート配下に、既存を壊さないように **新規作成 or 追記候補として生成**する。  
既存ファイルがある場合は **上書きせず**、`.generated` を付けて出力する。

### 1) 生成する推奨ツリー（固定）

    .
    ├─ src/
    │  └─ main/
    │     ├─ liberty/
    │     │  └─ config/
    │     │     └─ bootstrap.properties              (無ければ作成 / あれば .generated)
    │     └─ resources/
    │        └─ META-INF/
    │           └─ microprofile-config.properties    (無ければ作成 / あれば .generated)
    ├─ config/
    │  ├─ profiles/
    │  │  ├─ microprofile-config-dev.properties
    │  │  ├─ microprofile-config-stg.properties
    │  │  └─ microprofile-config-prod.properties
    │  └─ README-mp-config.md
    ├─ deploy/
    │  └─ k8s/
    │     ├─ secret-template.yaml
    │     └─ configmap-template.yaml                 (必要な場合のみ)
    └─ .env.example

> 既にプロジェクトの慣習（例：`src/main/liberty/config` が無い等）が見える場合は、  
> **最小限の推奨変更**として別案も併記する（質問はしない）。

***

## テンプレ内容（生成方針）

### A) `src/main/liberty/config/bootstrap.properties`

目的：Liberty 側の最低限の外枠。  
MP Config のキー自体は通常ここに置かないが、**プロファイル指定や外部化の導線**を置く。

生成ルール：

*   既に `bootstrap.properties` があるなら **触らず** `.generated` を出す
*   無ければ次を生成（コメント多め、値は空やプレースホルダ）：

**含める内容（最小）**

*   `# MP Config profile` の案内
*   `# 例: mp.config.profile は環境変数/システムプロパティでも指定可能` のメモ
*   Liberty 変数と混同しない注意書き

### B) `src/main/resources/META-INF/microprofile-config.properties`

目的：アプリ側デフォルト（dev 以外は空に近い）。  
生成ルール：

*   抽出キー一覧を **カテゴリ別**に並べる（後述）
*   値は以下の優先で入れる：
    *   `defaultValue` が抽出できた → その値
    *   既存 properties に値があった → その値をコメント付きで採用
    *   不明 → 空（`key=`）にする（値を勝手にでっち上げない）

**プロファイルについて**

*   1ファイル内 `%dev.key=` 記法を使う案をコメントで示す
*   ただし「プロファイル別雛形」は別ファイルでも生成する（ユーザー要望）

### C) `config/profiles/microprofile-config-{dev,stg,prod}.properties`

目的：環境別の差分を明示する雛形（運用設計に寄せる）。

生成ルール：

*   3ファイルとも同じキーセットを出し、値は原則空
*   **dev** には「ローカル実行向け」コメントを多めに
*   **stg/prod** には「Secrets/Env 経由推奨」コメントを多めに

> 注意：MP Config は標準ではこのファイル名を自動ロードしない。  
> そのため `README-mp-config.md` に以下の2案を必ず書く：
>
> 1.  `%dev.` 記法で 1ファイル運用
> 2.  ビルド/デプロイ時に対象ファイルを `META-INF/microprofile-config.properties` に配置する運用

### D) `.env.example`

目的：ローカルやコンテナでの環境変数注入の雛形。

生成ルール：

*   MP Config の **環境変数名マッピング**を併記する：
    *   基本：`foo.bar` → `FOO_BAR`（英大文字化＋区切りを `_`）
    *   `-` や `/` 等も `_` に寄せる（説明コメントを付ける）
*   値は空 or 既定値がある場合のみ例として入れる（ただし機密っぽいものは空）

### E) `deploy/k8s/secret-template.yaml`

目的：本番相当での Secret 注入の雛形。

生成ルール：

*   “機密候補キー”を自動判定して Secret 側に寄せる（推定）
    *   キー名に `password|passwd|secret|token|apikey|api_key|key|credential|private|cert` が含まれる（大小無視）
*   Secret の `stringData` を使い、値はプレースホルダ
*   併せて `envFrom` または個別 `env` 例をコメントで提示

### F) `deploy/k8s/configmap-template.yaml`（必要な場合のみ）

*   機密でないキーが一定数ある、またはプロファイル差分が大きい場合に生成
*   それ以外は Secret テンプレのみで十分なので生成しない（最小化）

### G) `config/README-mp-config.md`

目的：運用に迷わない “1枚ガイド”。

必ず含める：

*   抽出したキー一覧の出典と判定ルール
*   プロファイル運用（`MP_CONFIG_PROFILE` / `mp.config.profile`）
*   `microprofile-config.properties` と env/secret の優先順位イメージ
*   生成物の配置と、既存ファイルがある場合の扱い（上書きしない）

***

## キーの並べ方（読みやすさ重視）

キー一覧は以下の順でグルーピングして出力（自動）：

1.  `app.*` / `application.*` / `service.*`（アプリ一般）
2.  `http.*` / `rest.*` / `cors.*`（API）
3.  `db.*` / `datasource.*` / `jdbc.*`（DB）
4.  `mq.*` / `jms.*` / `kafka.*`（メッセージング）
5.  `auth.*` / `oidc.*` / `jwt.*`（認証認可）
6.  `metrics.*` / `health.*` / `tracing.*`（運用）
7.  その他（アルファベット順）

グループ推定できない場合は “その他” に入れる。

***

## 生成時の衝突ルール（安全第一）

*   同名ファイルが存在する場合は **上書きしない**
    *   例：`microprofile-config.properties.generated`
*   既存に追記が望ましい場合は、**差分パッチ案**を README に書く（自動編集はしない）
*   キーの重複がある場合：
    *   出典が異なる値を見つけたら両方をコメント付きで提示し、断定しない

***

## 出力フォーマット（固定）

1.  **探索結果サマリ**
    *   探索ルート
    *   ビルドツール判定（Maven/Gradle/不明）
    *   解析対象（言語/ファイル数の概算でOK）

2.  **設定キー一覧（必須/任意/既定値/出典/確度）**
    *   形式：箇条書き（テーブルにしない：コード混在時に崩れやすいため）
    *   例：
        *   `foo.bar` — required: ✅ / default: `baz` / source: annotation / confidence: high

3.  **生成ファイル一覧（パス + 役割）**

4.  **生成内容（主要ファイルの中身）**
    *   `bootstrap.properties`
    *   `microprofile-config.properties`
    *   `.env.example`
    *   `secret-template.yaml`
    *   `profiles/*` の例（全部は長ければ dev のみ全文＋他は差分）

5.  **運用メモ（README 相当の要点）**
    *   プロファイル運用
    *   Secret/Env への寄せ方
    *   次にやるべき検証（起動・代表ユースケース・外部接続）

6.  **追加質問（必要最小限）**
    *   例：`server.xml` が複数で Liberty サーバ名が確定できない等、生成先の判断に必須なときのみ

***

## “必要最小限”だけ質問してよい条件

自動で決められない場合に限り、次のどれかを1回だけ聞く：

*   `src/main/liberty/config/` が無く、代替候補が複数ある → **生成先ディレクトリ候補を提示して選んでもらう**
*   `microprofile-config.properties` が複数箇所に存在し、どれが正か判断不能 → **優先したい場所を選んでもらう**
*   リポジトリがモノレポでサブプロジェクトが複数 → **対象サブディレクトリを選んでもらう**

***

## 仕上げの一言（このコマンドの約束）

*   解析・生成は自動で行う
*   既存ファイルは壊さない（上書きしない）
*   不明点は **断定せず** 🟡要確認として残す
*   “動く設定”はあなたの環境依存なので、値は基本プレースホルダで安全に出す
