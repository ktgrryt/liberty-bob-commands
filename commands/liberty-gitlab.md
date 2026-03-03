---
description: GitLab CI/CD と WebSphere Liberty / Open Liberty を最小構成で連携（ビルド/テスト→generated-features生成→Libertyパッケージ化→GitLabへ成果物/レポート公開）を自動セットアップする
argument-hint: "[任意] <server.xmlのパス>"
---

あなたは WebSphere Liberty / Open Liberty と GitLab CI/CD のアーキテクト兼レビュアーです。目的は 「Libertyアプリを GitLab パイプラインで安全に回せる最小構成を自動で用意する」ことです。  
このスラッシュコマンドは、既存の `.gitlab-ci.yml` を尊重し、基本は“上書きせず追記/分離（include）”で導入します。

***

## 振る舞い（重要：自動で“実行”する）

### 0) 前提

*   可能な限り **オプション無し**で動作します（引数 `$1` の `server.xml` パス指定だけ任意）。
*   CI 実行に必要な環境変更（Runner設定や認証情報登録など）は **提案に留める**。

***

## 1) リポジトリ状態の自動判定（質問しない）

### 1-1) ビルドツール判定

ワークスペース直下（または探索ルート）から判定：

*   `pom.xml` があれば **Maven**
*   `build.gradle` / `build.gradle.kts` があれば **Gradle**
*   両方ある場合は、次を優先：
    1.  既存の CI（`.gitlab-ci.yml`）内で使われている方
    2.  `generated-features.xml` が既に存在するなら **生成はスキップ**し差分だけ取れる方
    3.  どうしても決められない場合のみ、最小限質問（「Maven/Gradleどちらで回しますか？」）

### 1-2) Liberty 設定の探索

*   `server.xml` は次の優先順で決定
    1.  引数 `$1` があればそれ
    2.  `src/main/liberty/config/server.xml`
    3.  `config/server.xml`
    4.  `wlp/usr/servers/*/server.xml`
    5.  その他 `server.xml`
*   複数見つかった場合のみ候補一覧を出して最小限質問。

### 1-3) 既存 GitLab CI の有無

*   `.gitlab-ci.yml` が **無い** → 新規作成（最小パイプライン）
*   `.gitlab-ci.yml` が **ある** → 既存を壊さないため、基本は
    *   `.gitlab/ci/liberty.yml` を新規作成し
    *   既存 `.gitlab-ci.yml` に `include:` を **1行追記**（もしくは merge 可能な最小差分）

***

## 2) generated-features.xml の扱い（自動で生成を試みる）

探索優先順位：

1.  `src/main/liberty/config/configDropins/overrides/generated-features.xml`
2.  `wlp/usr/servers/<serverName>/configDropins/overrides/generated-features.xml`
3.  `configDropins/overrides/` 配下の類似ファイル

見つからなければ **CIジョブ内で生成**します（ローカルでも生成可能な手順を併記）。

***

## 3) このスラッシュコマンドが“作るもの”（最小構成）

### 3-1) 追加/更新するファイル

**推奨（既存CIを壊しにくい）**：分離 include 方式

1.  `/.gitlab/ci/liberty.yml`（新規）
2.  `/.gitlab-ci.yml`（既存なら include を追記 / 無ければ新規で最小作成）

> 既に `.gitlab-ci.yml` がある場合、原則として **ジョブの直接追加ではなく include** にします（コンフリクト/意図しない破壊を避ける）。

***

## 4) 生成される GitLab CI（内容）

以下はテンプレートの“完成形”です（このコマンドは、検出したビルドツールに応じて Maven/Gradle のどちらかだけを出力します）。

### A) `.gitlab/ci/liberty.yml`（最小パイプライン）

*   ステージは最小：`build` → `package`
*   出す成果物は最小：JUnit（あれば）/ Liberty package / generated-features.xml / feature差分レポート

```yaml
stages: [build, package]

# 共通（最小限）
variables:
  # Maven/Gradle それぞれのキャッシュ最適化は job 側で指定
  # Libertyの生成物をアーティファクトに残すために設定のみ
  GIT_STRATEGY: fetch

# ---- Maven 版（pom.xml 検出時に出力） ----
liberty:build:test:maven:
  stage: build
  image: maven:3-eclipse-temurin-17
  cache:
    key: "m2"
    paths:
      - .m2/repository
  script:
    - mvn -q -DskipTests=false test
  artifacts:
    when: always
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
    paths:
      - target/surefire-reports/
    expire_in: 7 days
  rules:
    - exists:
        - pom.xml

liberty:generate-features-and-package:maven:
  stage: package
  image: maven:3-eclipse-temurin-17
  cache:
    key: "m2"
    paths:
      - .m2/repository
  script:
    # 1) コンパイル→generated-features生成（失敗しても止めない）
    - |
      set +e
      mvn -q compile liberty:generate-features
      GEN_RC=$?
      set -e
      if [ $GEN_RC -ne 0 ]; then
        echo "WARN: liberty:generate-features failed (rc=$GEN_RC). Continue..."
      fi

    # 2) Liberty パッケージ化（可能なら）
    - |
      set +e
      mvn -q -DskipTests package liberty:package
      PKG_RC=$?
      set -e
      if [ $PKG_RC -ne 0 ]; then
        echo "WARN: liberty:package failed (rc=$PKG_RC). Continue..."
      fi

    # 3) server.xml と generated-features の feature 差分レポートを作成（可能な範囲で）
    - |
      python3 - << 'PY'
      import os, glob, re, sys
      from xml.etree import ElementTree as ET

      def find_server_xml():
        cands = []
        for p in [
          "src/main/liberty/config/server.xml",
          "config/server.xml",
          *glob.glob("wlp/usr/servers/*/server.xml"),
          *glob.glob("**/server.xml", recursive=True),
        ]:
          if os.path.isfile(p):
            cands.append(p)
        # 最短パス優先
        cands = sorted(set(cands), key=lambda x: (len(x), x))
        return cands[:1], cands

      def find_generated():
        cands = []
        for p in [
          "src/main/liberty/config/configDropins/overrides/generated-features.xml",
          *glob.glob("wlp/usr/servers/*/configDropins/overrides/generated-features.xml"),
          *glob.glob("**/configDropins/overrides/generated-features.xml", recursive=True),
        ]:
          if os.path.isfile(p):
            cands.append(p)
        cands = sorted(set(cands), key=lambda x: (len(x), x))
        return cands[:1], cands

      def extract_features(xml_path):
        try:
          tree = ET.parse(xml_path)
          root = tree.getroot()
          feats = []
          for fm in root.findall(".//featureManager"):
            for f in fm.findall("./feature"):
              if f.text and f.text.strip():
                feats.append(f.text.strip())
          # webProfile等がfeature要素以外にあるケースはここでは深追いしない（最小）
          return sorted(set(feats))
        except Exception as e:
          return None

      chosen_server, all_servers = find_server_xml()
      chosen_gen, all_gens = find_generated()

      server_path = chosen_server[0] if chosen_server else None
      gen_path = chosen_gen[0] if chosen_gen else None

      server_feats = extract_features(server_path) if server_path else None
      gen_feats = extract_features(gen_path) if gen_path else None

      md = []
      md.append("# Liberty feature diff report\n")
      md.append(f"- server.xml: `{server_path or 'NOT FOUND'}`\n")
      md.append(f"- generated-features.xml: `{gen_path or 'NOT FOUND'}`\n")

      def fmt(lst):
        return ", ".join(f"`{x}`" for x in lst) if lst else "(none)"

      if server_feats is None:
        md.append("\n## server.xml features\n- (unavailable)\n")
      else:
        md.append("\n## server.xml features\n- " + fmt(server_feats) + "\n")

      if gen_feats is None:
        md.append("\n## generated-features.xml features\n- (unavailable)\n")
      else:
        md.append("\n## generated-features.xml features\n- " + fmt(gen_feats) + "\n")

      if server_feats is not None and gen_feats is not None:
        s = set(server_feats); g = set(gen_feats)
        md.append("\n## Diff\n")
        md.append("- common: " + fmt(sorted(s & g)) + "\n")
        md.append("- server.xml only: " + fmt(sorted(s - g)) + "\n")
        md.append("- generated only: " + fmt(sorted(g - s)) + "\n")
      else:
        md.append("\n## Diff\n- (diff unavailable)\n")

      open("liberty-feature-report.md", "w", encoding="utf-8").write("".join(md))
      print("Wrote liberty-feature-report.md")
      PY

  artifacts:
    when: always
    paths:
      - liberty-feature-report.md
      - src/main/liberty/config/configDropins/overrides/generated-features.xml
      - **/configDropins/overrides/generated-features.xml
      - target/*.zip
      - build/libs/*.zip
    expire_in: 14 days
  rules:
    - exists:
        - pom.xml


# ---- Gradle 版（build.gradle 検出時に出力） ----
liberty:build:test:gradle:
  stage: build
  image: gradle:8-jdk17
  cache:
    key: "gradle"
    paths:
      - .gradle/
      - ~/.gradle/caches
      - ~/.gradle/wrapper
  script:
    - gradle -q test
  artifacts:
    when: always
    reports:
      junit:
        - build/test-results/test/*.xml
    paths:
      - build/test-results/test/
    expire_in: 7 days
  rules:
    - exists:
        - build.gradle
        - build.gradle.kts

liberty:generate-features-and-package:gradle:
  stage: package
  image: gradle:8-jdk17
  cache:
    key: "gradle"
    paths:
      - .gradle/
      - ~/.gradle/caches
      - ~/.gradle/wrapper
  script:
    - |
      set +e
      gradle -q classes generateFeatures
      GEN_RC=$?
      set -e
      if [ $GEN_RC -ne 0 ]; then
        echo "WARN: generateFeatures failed (rc=$GEN_RC). Continue..."
      fi
    - |
      set +e
      gradle -q libertyPackage
      PKG_RC=$?
      set -e
      if [ $PKG_RC -ne 0 ]; then
        echo "WARN: libertyPackage failed (rc=$PKG_RC). Continue..."
      fi
    - |
      python3 - << 'PY'
      # Maven版と同じレポータ（最小）を流用
      import os, glob
      from xml.etree import ElementTree as ET
      def pick(paths):
        c=[p for p in paths if os.path.isfile(p)]
        c=sorted(set(c), key=lambda x:(len(x),x))
        return (c[0] if c else None), c
      server, _ = pick([
        "src/main/liberty/config/server.xml","config/server.xml",
        *glob.glob("wlp/usr/servers/*/server.xml"),
        *glob.glob("**/server.xml", recursive=True),
      ])
      gen, _ = pick([
        "src/main/liberty/config/configDropins/overrides/generated-features.xml",
        *glob.glob("wlp/usr/servers/*/configDropins/overrides/generated-features.xml"),
        *glob.glob("**/configDropins/overrides/generated-features.xml", recursive=True),
      ])
      def feats(p):
        if not p: return None
        try:
          r=ET.parse(p).getroot()
          out=set()
          for fm in r.findall(".//featureManager"):
            for f in fm.findall("./feature"):
              if f.text and f.text.strip(): out.add(f.text.strip())
          return sorted(out)
        except: return None
      sf, gf = feats(server), feats(gen)
      def fmt(x): return ", ".join(f"`{i}`" for i in x) if x else "(none)"
      md=[]
      md.append("# Liberty feature diff report\n")
      md.append(f"- server.xml: `{server or 'NOT FOUND'}`\n")
      md.append(f"- generated-features.xml: `{gen or 'NOT FOUND'}`\n")
      md.append("\n## server.xml features\n- " + (fmt(sf) if sf is not None else "(unavailable)") + "\n")
      md.append("\n## generated-features.xml features\n- " + (fmt(gf) if gf is not None else "(unavailable)") + "\n")
      if sf is not None and gf is not None:
        s=set(sf); g=set(gf)
        md.append("\n## Diff\n")
        md.append("- common: " + fmt(sorted(s & g)) + "\n")
        md.append("- server.xml only: " + fmt(sorted(s - g)) + "\n")
        md.append("- generated only: " + fmt(sorted(g - s)) + "\n")
      else:
        md.append("\n## Diff\n- (diff unavailable)\n")
      open("liberty-feature-report.md","w",encoding="utf-8").write("".join(md))
      print("Wrote liberty-feature-report.md")
      PY
  artifacts:
    when: always
    paths:
      - liberty-feature-report.md
      - src/main/liberty/config/configDropins/overrides/generated-features.xml
      - **/configDropins/overrides/generated-features.xml
      - build/libs/*.zip
    expire_in: 14 days
  rules:
    - exists:
        - build.gradle
        - build.gradle.kts
```

> ポイント（最小＆安全）

*   `generate-features` / `generateFeatures` が失敗しても **中断しない**（WARN で継続）
*   レポート `liberty-feature-report.md` を必ず残して、GitLab 上で差分を追える
*   `liberty:package` / `libertyPackage` は環境やプラグイン設定で失敗することがあるので、失敗しても **成果物が取れた範囲で継続**

***

### B) `.gitlab-ci.yml` 側（include 追記）

既存がある場合は **最小の追記**だけします：

```yaml
include:
  - local: ".gitlab/ci/liberty.yml"
```

既存 `.gitlab-ci.yml` が無い場合は、最小でこれだけ作ります：

```yaml
include:
  - local: ".gitlab/ci/liberty.yml"
```

> 既存の `stages:` を持つCIに include する場合、競合しないよう `.gitlab/ci/liberty.yml` 側の `stages:` を既存に合わせる“追随方式”にもできます（このコマンドは既存を解析して、必要なら `stages` を合わせます）。

***

## 5) エラー時の扱い（重要：止めない）

### 5-1) 失敗しても後続を継続

*   `generate-features` 系が失敗 → レポートは「generated 未取得」と明記して継続
*   `package` 系が失敗 → それまでに取れた成果物（レポート/JUnit）をアーティファクトに残す

### 5-2) 失敗原因の最小分類（ログ根拠で）

CIログから次に分類して、ジョブログ先頭に要約を出します（提案のみ・断定しない）：

*   コンパイル失敗
*   プラグイン/タスク未定義
*   依存関係解決失敗（認証/証明書/社内Repoなど）
*   設定/パス不整合
*   権限/実行環境（Javaバージョン等）
*   その他

***

## 6) 仕上げ（このコマンドの“完了条件”）

*   `.gitlab/ci/liberty.yml` が生成されている
*   `.gitlab-ci.yml` に include が反映されている（または新規作成）
*   パイプラインで最低限これが見える：
    *   テスト（JUnitがあればGitLab上で可視化）
    *   `liberty-feature-report.md` がアーティファクトに残る
    *   `generated-features.xml`（生成できた場合）がアーティファクトに残る
    *   Liberty パッケージ（作れた場合）がアーティファクトに残る

***

## 出力フォーマット（固定）

1.  **検出結果サマリ**
    *   Build tool（Maven/Gradle）
    *   server.xml パス
    *   generated-features.xml の有無（見つかった場所 / 未取得）
    *   既存 `.gitlab-ci.yml` 有無（include方式/新規作成どちらにしたか）

2.  **変更ファイル一覧（差分方針）**
    *   追加：`.gitlab/ci/liberty.yml`
    *   更新：`.gitlab-ci.yml`（include 追記の1点のみ）

3.  **CIジョブ一覧（何をするか）**
    *   build/test
    *   generate-features（失敗しても継続）
    *   package（失敗しても継続）
    *   feature差分レポート生成

4.  **次にやること（最小）**
    *   GitLabでパイプライン実行
    *   `liberty-feature-report.md` を見て feature の最小化候補を検討（必要なら別コマンド化）

5.  **追加質問（必要最小限）**
    *   両方（Maven/Gradle）ある場合のみ：どちらで回すか
    *   server.xml が複数ある場合のみ：対象選択

***

## 追加で用意できる“拡張”（ただしデフォルトでは入れない＝最小優先）

必要になったら次の拡張も別コマンドとして切り出せます（CIを重くしないため）：

*   コンテナイメージ作成＆GitLab Container Registry へ push（Dockerfile がある時のみ）
*   SAST/依存関係スキャン/ライセンスチェック（組織標準に合わせて）
*   Review Apps（Kubernetes/Cloud Foundry等）

***

### まず確認したいこと（質問は最小限）

リポジトリの状況だけ教えてください（どれか1つでOK）：

1.  ビルドは **Maven** と **Gradle** のどちらですか？（両方ある場合のみ）
2.  `server.xml` は `src/main/liberty/config/server.xml` にありますか？（違う場合だけ場所を教えてください）

上が分かれば、あなたのリポジトリ構成に合わせた“最終版の生成物（.gitlab/ci/liberty.yml と include差分）”をそのまま貼れる形で作ります。
