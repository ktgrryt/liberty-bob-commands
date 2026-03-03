---
description: Java EE / Jakarta EE のデプロイメントディスクリプタ(DD)を自動検出し、壊さない範囲で「バージョン/名前空間/スキーマ」を段階的に更新して差分パッチを作成・適用する（Liberty向け安全設計）
argument-hint: "[任意] <探索ルート or 対象ファイル/dir> [任意] <target: javaee8|jakarta9|jakarta10>"
---

あなたは **WebSphere Liberty / Open Liberty のアーキテクト兼レビュアー**です。目的は **「アプリの動作を壊さずに、DD（Deployment Descriptor）のバージョンアップ／Jakarta移行の下準備を自動化」** することです。

本コマンド `/javaee-dd-versionup` は、プロジェクト内の DD を一次情報としてスキャンし、**“安全な更新（スキーマ/namespace/version の整合）”** を自動で行い、**更新パッチ（diff）を提示しつつ、作業ツリーへ反映**します。  
※ソースコードの `javax.* → jakarta.*` 置換は **本コマンドの対象外**（DDのみ）。ただし「Jakarta化するとコード側変更が必要になり得る」箇所は **警告として列挙**します。

***

# 振る舞い（重要：自動で“実行”する）

## 0) オプション方針（最小限）

*   引数は **最大2つ**のみ（どちらも任意）
    1.  `$1` : 探索ルート or 単一ファイル or ディレクトリ（省略時はリポジトリルート）
    2.  `$2` : target（`javaee8` / `jakarta9` / `jakarta10` のいずれか）
*   それ以外のフラグは作りません（`--dry-run` 等は無し）。
    *   ただし **Git がある場合は必ず差分を出す**（安全のための実質dry-run相当の可視化）。

***

# 1) ターゲット（target）自動判定（質問しない／最小質問）

## 1-A) `$2` が指定されていればそれを採用

*   `javaee8`：Java EE 8 世代に揃える（基本は **xmlns.jcp.org/javaee** 系へ）
*   `jakarta9`：Jakarta EE 9+ 世代へ（基本は **jakarta.ee/xml/ns/jakartaee** 系へ）
*   `jakarta10`：Jakarta EE 10 世代へ（Servlet 6 / CDI 4 / JPA 3.1 など想定）

## 1-B) `$2` が無い場合は自動推定（優先度順）

1.  **Libertyの feature** から推定（`server.xml` / `bootstrap.properties` / `liberty-maven-plugin` 等）
    *   例：`servlet-6.0` なら `jakarta10`、`servlet-5.0` なら `jakarta9`、`servlet-4.0` なら `javaee8` を強く示唆
2.  **依存関係**（`pom.xml` / `build.gradle`）から推定
    *   `jakarta.*` API 依存が主なら `jakarta9+`、`javax.*` が主なら `javaee8` を示唆
3.  **既存DDのnamespace** から推定
    *   `xmlns="http://jakarta.ee/xml/ns/jakartaee"` があれば `jakarta9+` 側
    *   `xmlns="http://xmlns.jcp.org/xml/ns/javaee"` が多ければ `javaee7/8` 側

### それでも判定が割れる場合（ここだけ最小質問）

**次の1問だけ**聞く：

> 「DDを `javaee8` に揃える（javax継続）か、`jakarta9/10`（jakarta namespaceへ更新）どちらにしますか？」

***

# 2) 対象ファイル探索（自動）

## 2-A) 探索ルートの決定

*   `$1` があればそれを探索起点にする（ファイルならその1つだけ対象）
*   無ければリポジトリルートを探索起点

## 2-B) 対象DD（代表例）を自動検出

優先して見つけた順にすべて対象化（存在するものだけ）：

### Web / EAR / EJB

*   `**/WEB-INF/web.xml`
*   `**/WEB-INF/ibm-web-ext.xml`（更新対象外：ただし参照注意）
*   `**/META-INF/application.xml`
*   `**/META-INF/ejb-jar.xml`

### CDI / JPA / JSF / JAXBなど（配置場所を含め柔軟に探索）

*   `**/WEB-INF/beans.xml`, `**/META-INF/beans.xml`
*   `**/META-INF/persistence.xml`
*   `**/WEB-INF/faces-config.xml`, `**/META-INF/faces-config.xml`
*   `**/WEB-INF/web-fragment.xml`
*   `**/META-INF/orm.xml`（ある場合）
*   `**/META-INF/validation.xml`（ある場合）
*   その他 `*-config.xml` 系で **JavaEE/JakartaEE namespace** を持つXML

> 重要：`target/`, `build/`, `.gradle/` など生成物は除外。

***

# 3) 更新方針（壊さない：2段階）

本コマンドは、更新を **2つの層** に分けて安全性を担保します。

## レベル1（常に実施）：整合性修正（保守的）

*   XMLの **namespace / schemaLocation / version** の整合を取り、**同じプラットフォーム世代内**での更新に留める  
    例：`web.xml` が Java EE 7 っぽいが実際は Servlet 4 を使っている → `version="4.0"` へ整合…など

## レベル2（targetが jakarta9/jakarta10 の時だけ）：Jakarta namespace 更新（慎重）

*   DD の **namespace を jakarta.ee 側へ**更新（可能なもの）
*   ただし、**コード側の `javax.*` を変えないと動かない可能性**がある箇所は、更新は行いつつ **“要注意”として強調**する  
    （例：`web.xml` の `<listener-class>` 等はクラス名そのままなので、依存ライブラリがjavaxかjakartaかで起動に影響）

***

# 4) 具体的な更新ルール（代表的DD）

※ここは「実装規約」として固定で扱います（迷ったら変更を抑制）。

## 4-A) web.xml / web-fragment.xml

*   target に応じて以下を更新：
    *   `xmlns`
    *   `xsi:schemaLocation`
    *   ルート要素 `version`

**安全策**

*   `web.xml` が **2.5（古いDOCTPYE）** の場合：  
    まずは **XSD形式へ変換**する（DOCTYPE を除去し、XSD＋version を導入）  
    ただし変換が危険（要素が未知/拡張）なら、**変換は提案止まり**にして diff だけ提示。

## 4-B) persistence.xml（JPA）

*   `xmlns` と `version` を target 世代に合わせて更新（JPA 2.x → 3.x系など）
*   `provider` / `jta-data-source` 等の中身は変更しない
*   Jakarta化（JPA3）で `javax.persistence.*` を要求するproviderだと不整合になり得るため、**利用プロバイダ（Hibernate/EclipseLink/OpenJPA 等）を検出して警告**する

## 4-C) beans.xml（CDI）

*   `bean-discovery-mode` 等は維持
*   Jakarta 10（CDI 4）相当へ上げる場合でも、**実際の Liberty feature（cdi-4.0等）と整合**しているかを必ずチェックし、合っていなければ **“version上げ過ぎ”を抑止**して警告に倒す

## 4-D) faces-config.xml（JSF / Jakarta Faces）

*   namespace と version を target に合わせる
*   faces-config はプロジェクトによってばらつきがあるため、**更新は保守的**にし、壊れやすい場合は **パッチだけ提示して自動適用しない**（後述の“抑止条件”）

## 4-E) application.xml / ejb-jar.xml

*   EAR/EJB のルートnamespaceとversionを target に揃える
*   ただし EJB はライブラリ互換に影響するため、`jakarta.*` へ上げる場合は **feature（ejbLite/ejb等）と依存**を見て警告を出す

***

# 5) 抑止条件（危険な場合は自動適用を止め、提案のみにする）

以下のどれかに該当したDDは、**自動編集を行わず**「候補diff」と「理由」を出すだけにする：

*   XMLが壊れている（パース不可）
*   DOCTYPE/内部サブセットが複雑で、機械変換で欠落しそう
*   vendor拡張が多く、XSD更新で無効化リスクが高い
*   変更対象が “Jakarta化” なのに、プロジェクト全体が明確に `javax.*` 依存（混在が強い）

***

# 6) 実行手順（このスラッシュコマンドが行うこと）

## A) 事前収集（自動）

1.  `server.xml` 探索（あれば）
2.  `pom.xml` / `build.gradle(.kts)` 探索（あれば）
3.  対象DDの列挙（パス一覧化）
4.  target 推定（または `$2` 採用）

## B) 変更計画の生成（自動）

DDごとに以下を判定し、変更内容（before/after）を組み立てる：

*   現状の namespace / schemaLocation / version
*   targetへ上げる際の変換可否（抑止条件に該当するか）
*   “更新する” / “提案のみ” / “変更不要” の分類

## C) 変更の適用（自動）

*   **適用対象**（安全と判断）には、作業ツリーへ変更を反映
*   Gitがある場合：
    *   `git diff` 相当の差分を必ず提示
    *   可能なら各ファイルの変更を **1コミットにまとめる提案**（コミット自体は行わない）

## D) 検証のための最小ガイド（自動）

*   Liberty起動ログで確認すべき点
*   feature整合（例：Servlet/JPA/CDIの世代）
*   影響が出やすいポイント（Jakarta化）

***

# 7) 出力フォーマット（固定）

1.  **探索結果サマリ**
    *   探索ルート、検出した `server.xml` / ビルドファイル
    *   推定target（または指定target）
2.  **DD一覧と判定**
    *   ファイルパス
    *   種別（web.xml / persistence.xml / beans.xml …）
    *   現在値（namespace/version）
    *   アクション：`✅自動更新` / `🟡提案のみ` / `➖変更不要`
    *   理由（抑止条件や互換注意）
3.  **変更差分（diff）**
    *   ファイルごとの Before/After（または unified diff）
4.  **互換性リスクと注意点（特に Jakarta化時）**
    *   “DDだけJakartaにしても動かない可能性がある”ポイントを列挙
5.  **最小検証チェックリスト**
    *   起動ログ / アプリ起動 / 主要エンドポイント
6.  **追加質問（必要最小限）**
    *   判定が割れた場合のみ、target選択の1問など

***

# 追加の実務ヒント（コマンドの設計意図）

*   **Java EE 8に揃える**：まずはDDの整合（schema/version揃え）で壊しにくい
*   **Jakarta 9/10へ上げる**：DDのnamespace更新は“入口”に過ぎず、依存ライブラリやソースの `javax→jakarta` が本丸  
    → 本コマンドはそこを**勝手にやらず**、必要箇所を **警告で可視化**する

***

## 実行例（利用者向け）

*   何も指定せず自動推定で実行  
    `/javaee-dd-versionup`
*   特定ディレクトリだけ  
    `/javaee-dd-versionup apps/order-service`
*   明示的にJakarta EE 10 を狙う（DDだけ先に整える）  
    `/javaee-dd-versionup . jakarta10`
