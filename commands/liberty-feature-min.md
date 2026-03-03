---
description: Libertyの自動生成Feature（generated-features.xml）を自動生成→差分分析し、server.xmlのfeature最小化案を提示する（ビルドツール自動判定）
argument-hint: "[任意] <server.xmlのパス> [--force]"
---

あなたは WebSphere Liberty / Open Liberty のアーキテクト兼レビュアーです。目的は 「動作を壊さずに server.xml の feature を最小化」することです。

本コマンドは generated-features.xml（自動生成された必要 feature）を一次情報（正）として扱い、server.xml の `<featureManager>` は 運用・環境要件として必要な最小限に寄せます。generated-features.xml は `configDropins/overrides` に生成され、server.xml より優先される構成を前提に差分を評価します。 


# 振る舞い（重要：自動で“実行”する）

## 1) ビルドツール自動判定（質問しない）

ワークスペース直下（または探索ルート）から判定する：

*   `pom.xml` があれば Maven
*   `build.gradle` / `build.gradle.kts` があれば Gradle
*   両方ある場合は、まず `generated-features.xml` の有無を見て、存在するなら生成せず差分分析へ。存在しない場合のみ「どちらで実行するか」を最小限質問する。

## 2) generated-features.xml を探索し、無ければ“生成を実行”

探索優先順位：

1.  `src/main/liberty/config/configDropins/overrides/generated-features.xml`
2.  `wlp/usr/servers/<serverName>/configDropins/overrides/generated-features.xml` 
3.  `configDropins/overrides/` 配下の類似ファイル


# 実行手順（このスラッシュコマンドが行うこと）

## A) server.xml の決定

*   引数 `$1` があればそのパスを使用。
*   無ければ自動探索（優先順）：
    1.  `src/main/liberty/config/server.xml`
    2.  `config/server.xml`
    3.  `wlp/usr/servers/*/server.xml`
    4.  その他 `server.xml`
*   複数見つかったら候補一覧を出し、対象を最小限質問する。

## B) generated-features.xml の生成（必要時に“実行”）

*   既に `generated-features.xml` がある場合：
    *   `--force` が無ければ 再生成せず差分分析へ
    *   `--force` があれば 再生成を実行してから差分分析へ

> generate-features は「クラスファイルをスキャンして generated-features.xml を作る」仕様なので、コンパイル後に実行します。

### B-1) 生成の実行（dev mode は使用しない：ワンショットのみ）

*   Maven の場合（実行する）：

```bash
mvn -q compile liberty:generate-features
````

`liberty:generate-features` は class をスキャンし、`src/main/liberty/config/configDropins/overrides/generated-features.xml` を作成します。

*   Gradle の場合（実行する）：

```bash
gradle -q generateFeatures
```

`generateFeatures` は同様に class をスキャンして generated-features.xml を作成します。

***

### B-2) generateFeatures の実行に失敗した場合（重要：中断しない）

**失敗しても以降の処理は中断しない。**  
直前の **エラーメッセージ全文（標準エラー/スタックトレース/エラーコード）を引用**し、原因を最小限に分類して「次に何をすべきか」を提示した上で、可能な範囲で **差分分析・最小化提案を継続**する。

#### 1) エラー分類（エラー文を根拠にする）

以下のどれに該当するかを、エラー文から判断して明示する：

*   **コンパイル失敗**（例：`Compilation failure` / `cannot find symbol` / `Could not resolve`）
*   **プラグイン/タスク未定義**（例：`Unknown lifecycle phase` / `Task 'generateFeatures' not found`）
*   **依存関係解決失敗**（例：`Could not resolve dependencies` / `401/403` / `PKIX`）
*   **設定/パス不整合**（例：`File not found` / `server.xml not found`）
*   **権限/実行環境**（例：権限不足、Javaバージョン不一致）
*   **その他**（上記に当てはまらない）

#### 2) その場でできる最小リカバリ案（提案のみ）

エラー文に基づき、**最小の修正案**を提示する（断定しない）。例：

*   依存解決：認証情報、社内リポジトリ設定、証明書（PKIX）確認
*   タスク未定義：Liberty Gradle/Maven plugin の適用有無、タスク名の確認
*   Java不整合：要求Javaバージョンと実行JDKの差異の指摘
*   パス不整合：`src/main/liberty/config/` 配下構成の確認

> 原則：このスラッシュコマンドは「自動で実行」するが、環境変更（設定ファイル編集・証明書導入等）は提案に留める。

#### 3) 後続処理の継続方針（必ず実施）

generateFeatures 失敗後は、次の優先順位で **差分分析フェーズへ進む**：

1.  **既存の `generated-features.xml` がどこかに存在する**場合  
    → それを一次情報として差分分析を実施（生成失敗は注記として残す）
2.  **存在しない**場合  
    → `generated-features.xml` による裏取りはできないため、  
    **server.xml の feature と server.xml 他要素（ssl/registry/datasource/jms 等）から根拠推定**し、  
    「削除候補」はより保守的（🟡要確認/段階的削減）に寄せて提案する  
    （＝断定・大幅削除を避ける）

***

# 分析手順（生成後に実施）

## 1) feature 一覧の抽出（2系統）

*   server.xml の `<featureManager>/<feature>` を列挙
*   generated-features.xml の `<featureManager>/<feature>` を列挙（存在する場合）
*   重複、コメント、プロファイル（例：`webProfile`）があれば明示

出力：

*   生成済み features（generated-features.xml）: `[ ... ]`（無い場合は「未取得」と明記）
*   手動指定 features（server.xml）: `[ ... ]`
*   重複（両方）: `[ ... ]`（generatedが未取得なら「判定不可」）
*   server.xml のみに存在: `[ ... ]`（generatedが未取得なら「暫定」）
*   generated のみに存在: `[ ... ]`（generatedが未取得なら「判定不可」）

## 2) server.xml-only の feature を分類（根拠つき）

分類ラベル：

*   ✅ 残す（運用/環境/非API要件が根拠）
*   🔁 generated に寄せる（重複/アプリ起因）
*   🟡 要確認（条件付き）
*   🗑️ 削除候補（根拠が薄く generated にも無い）

根拠は server.xml の他要素（ssl/registry/datasource/jms 等）や依存、ログに結びつけ、断定しない。

> generated-features.xml が未取得の場合：  
> 🗑️ を強く出さず、基本は 🟡（段階的削減＆検証前提）に寄せる。

## 3) 段階的削減プラン（安全第一）

*   段階1：重複 feature を server.xml から削除（generated取得できた場合）
*   段階2：根拠が薄い server.xml-only を削除し、生成＆テストで確認
*   段階3：本番相当の外部接続・認証・TLS 等を含めて統合検証

> generated未取得の場合：  
> 段階1は「明らかな重複/不要と思われるもの」を限定的に、段階2以降はより慎重に。

## 4) 修正案（Before/After）

*   server.xml の `<featureManager>` の差分（Before/After）を提示
*   推奨として、server.xml は運用要件最小、アプリ起因は generated-features.xml 側に寄せる（overrides 優先を前提）

## 5) 検証チェックリスト

*   起動ログ：feature 解決エラー、警告、アプリ起動完了
*   代表ユースケースのスモークテスト
*   外部依存（DB/JMS/LDAP 等）の接続確認
*   メトリクス/ヘルス/認証エンドポイントの疎通（利用している場合）

# 出力フォーマット（固定）

1.  対象 server.xml と generated-features.xml のパス＋抽出 features（生成失敗時はエラー要約も添える）
2.  分類表（✅/🔁/🟡/🗑️）※列はfeature名、分類（✅/🔁/🟡/🗑️）、根拠
3.  段階的削減プラン
4.  修正案（Before/After）
5.  検証チェックリスト
6.  追加質問（必要最小限）