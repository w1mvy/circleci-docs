---
layout: classic-docs
title: "依存関係のキャッシュ"
short-title: "依存関係のキャッシュ"
description: "依存関係のキャッシュ"
categories:
  - optimization
order: 50
---

キャッシュは、以前のジョブの高コストなフェッチ操作から取得したデータを再利用することで、CircleCI のジョブを効果的に高速化します。

- 目次
{:toc}

ジョブを 1 回実行すると、以降のジョブ インスタンスでは同じ処理をやり直す必要がなくなり、その分高速化されます。

![キャッシュのデータ フロー]({{ site.baseurl }}/assets/img/docs/caching-dependencies-overview.png)

キャッシュは、Yarn、Bundler、Pip などの**パッケージ依存関係管理ツール**と共に使用すると特に有効です。 キャッシュから依存関係を復元することで、`yarn install` などのコマンドを実行するときに、ビルドごとにすべてを再ダウンロードするのではなく、新しい依存関係をダウンロードするだけで済むようになります。

<div class="alert alert-warning" role="alert">
<b>警告:</b> 異なる Executor 間 (たとえば、Docker と Machine、Linux、Windows、MacOS の間、または CircleCI イメージとそれ以外のイメージの間) でファイルをキャッシュすると、ファイル パーミッション エラーまたはパス エラーが発生することがあります。 これらのエラーは、ユーザーが存在しない、ユーザーの UID が異なる、パスが存在しないなどの理由で発生します。 異なる Executor 間でファイルをキャッシュする場合は、特に注意してください。
</div>

## キャッシュ構成の例
{:.no_toc}

キャッシュ キーは簡単に構成できます。 以下の例では、`pom.xml` のチェックサムとカスケード フォールバックを使用して、変更があった場合にキャッシュを更新しています。

{% raw %}
```yaml
    steps:

      - restore_cache:
         keys:
           - m2-{{ checksum "pom.xml" }}
           - m2- # チェックサムが失敗した場合に使用されます
```
{% endraw %}

## はじめに
{:.no_toc}

CircleCI 2.0 では依存関係キャッシュの自動化を利用できません。このため、最適なパフォーマンスを得るには、キャッシュ戦略を計画して実装することが重要です。 2.0 では、キャッシュを手動で構成し、より高度な戦略を立て、きめ細かに制御することができます。

ここでは、キャッシュの手動構成、選択した戦略のコストとメリット、およびキャッシュに関する問題を回避するためのヒントについて説明します。 **メモ:** CircleCI 2.0 のジョブ実行に使用される Docker イメージは、サーバー インフラストラクチャに自動的にキャッシュされます (可能な場合)。

Docker イメージの未変更レイヤーを再利用するプレミアム機能を有効にする方法については、「[Docker レイヤー キャッシュの有効化]({{ site.baseurl }}/2.0/docker-layer-caching/)」を参照してください。

## 概要
{:.no_toc}

キャッシュは、キーに基づいてファイルの階層を保存します。 キャッシュを使用してデータを保存するとジョブが高速に実行されますが、キャッシュ ミス (ゼロ キャッシュ リストア) が起きた場合でも、ジョブは正常に実行されます。 たとえば、`npm` パッケージ ディレクトリ (`node_modules`) をキャッシュする場合は、初めてジョブを実行するときに、すべての依存関係がダウンロードされてキャッシュされます。キャッシュが有効な限り、次回そのジョブを実行するときにキャッシュが使用され、ジョブ実行が高速化されます。

キャッシュは、信頼性の確保 (古いキャッシュや不適切なキャッシュを使用しない) と最大限のパフォーマンス (すべてのビルドで完全にキャッシュを使用する) のどちらを優先するかを考慮して構成します。

通常は、ビルドが壊れる危険を冒したり、古い依存関係を使用して高速にビルドするよりも、信頼性の維持を優先した方が安全です。 このため、高い信頼性を確保しつつ、最大限のパフォーマンスを得られるようにバランスを取ることが理想的と言えます。

## ライブラリのキャッシュ

ジョブ実行中にキャッシュすることが最も重要な依存関係は、プロジェクトが依存するライブラリです。 たとえば、Python なら `pip`、Node.js なら `npm` でインストールされるライブラリをキャッシュします。 さまざまな言語の依存関係管理ツール (`npm`、`pip` など) には、依存関係がインストールされるパスがそれぞれ指定されています。 お使いのスタックの仕様については、各言語ガイドおよび[デモ プロジェクト](https://circleci.com/ja/docs/2.0/demo-apps/)を参照してください。

プロジェクトに明示的に必要でないツールは、Docker イメージに保存するのが理想的です。 CircleCI のビルド済み Docker イメージには、そのイメージが対象としている言語を使用してプロジェクトをビルドするための汎用ツールがプリインストールされています。 たとえば、`circleci/ruby:2.4.1` イメージには git、openssh-client、gzip などの便利なツールがプリインストールされています。

![依存関係のキャッシュ]({{ site.baseurl }}/assets/img/docs/cache_deps.png)

## ワークフローでのキャッシュへの書き込み

同じワークフロー内のジョブどうしはキャッシュを共有できます。 このため、複数のワークフローの複数のジョブにまたがってキャッシュを実行すると、競合状態が発生する可能性があります。

キャッシュは書き込み後に変更不可です。つまり、`node-cache-master` などの特定のキーにキャッシュが書き込まれると、そのキーに再度書き込むことはできません。 たとえば、3 個のジョブを含むワークフローがあるとします。この中で、Job3 は Job1 と Job2 に依存しています ({Job1, Job2} -> Job3)。 これらは、すべて同じキャッシュ キーに対して読み書きを行います。

このワークフローの実行中、Job3 は Job1 または Job2 によって書き込まれたキャッシュを使用します。 キャッシュは変更不可なので、どちらかのジョブによって最初に保存されたキャッシュが使用されます。 結果が確定的ではなく、その時々によって結果が異なるため、通常、この動作は好ましくありません。 これを確定的なワークフローにするには、ジョブの依存関係を変更します。Job1 と Job2 で別のキャッシュに書き込み、Job3 ではいずれかのキャッシュから読み込みます。または、一方向の依存関係を指定します (Job1 -> Job2 ->Job3)。

{% raw %}
`node-cache-{{ checksum "package-lock.json" }}
            `{% endraw %} のような動的キーを使用して保存を行い、`node-cache-` のようなキーの部分一致を使用して復元を行うような、より複雑なジョブのケースもあります。 競合状態が発生する可能性がありますが、詳細はケースによって異なります。 たとえば、ダウンストリーム ジョブがアップストリーム ジョブのキャッシュを使用して最後に実行されるような場合です。

ジョブ間でキャッシュを共有している場合に発生する競合状態もあります。 Job1 と Job2 の間に依存関係がないワークフローを例に考えます。 Job2 は Job1 によって保存されたキャッシュを使用します。 Job1 がキャッシュの保存を報告したとしても、Job2 でキャッシュを正常に復元できることもあれば、キャッシュが見つからないと報告することもあります。 また、Job2 が以前のワークフローからキャッシュを読み込むこともあります。 その場合は、Job1 がキャッシュを保存する前に、Job2 がキャッシュを読み込もうとすることになります。 この問題を解決するには、ワークフローの依存関係 (Job1 -> Job2) を作成します。 こうすることで、Job1 の実行が終了するまで、Job2 の実行を待機させることができます。

## キャッシュの復元

CircleCI では、`restore_cache` ステップにリストされているキーの順番でキャッシュが復元されます。 各キャッシュ キーはプロジェクトの名前空間にあり、プレフィックスが一致すると取得されます。 最初に一致したキーのキャッシュが復元されます。 複数の一致がある場合は、最も新しく生成されたキャッシュが使用されます。

次の例では、2 つのキーが提供されています。

{% raw %}
```yaml
    steps:

      - restore_cache:
          keys:
            # この package-lock.json のチェックサムに一致するキャッシュを検索します
            # このファイルが変更されている場合、このキーは失敗します
            - v1-npm-deps-{{ checksum "package-lock.json" }}
            # 任意のブランチから使用される、最も新しく生成されたキャッシュを検索します
            - v1-npm-deps-
```
{% endraw %}

2 つ目のキーは最初のキーよりも特定度が低いため、現在の状態と最も新しく生成されたキャッシュとの間に差がある可能性が高くなります。 依存関係ツールを実行すると、古い依存関係が検出されて更新されます。 これを**部分キャッシュ リストア**と言います。

上のキャッシュ キーの使用方法について、詳しく見ていきましょう。

Each line in the `keys:` list all manage *one cache* (each line does **not** correspond to its own cache). この例でリストされているキー

{% raw %}
(`v1-npm-deps-{{ checksum "package-lock.json" }}`{% endraw %} および `v1-npm-deps-`) は、**単一**のキャッシュを表しています。 キャッシュの復元が必要になると、まず (最も特定度の高い) 最初のキーに基づいてキャッシュがバリデーションされ、次に他のキーを順に調べて、他のキャッシュ キーに変更があるかどうかが確認されます。

ここでは、最初のキーによって `package-lock.json` ファイルのチェックサムが文字列 `v1-npm-deps-` に連結されます。コミットでこのファイルが変更されている場合は、新しいキャッシュ キーが調べられます。

次のキーには動的コンポーネントが連結されていません。これは静的な文字列 `v1-npm-deps-` です。 キャッシュを手動で無効にするには、`config.yml` ファイルで `v1` を `v2` にバンプします。 これで、キャッシュ キーが新しい `v2-npm-deps` になり、新しいキャッシュの保存がトリガーされます。

### モノレポ (モノリポ) でのキャッシュの使用

モノレポでキャッシュを活用する際のアプローチは数多くあります。 ここで紹介するアプローチは、モノレポのさまざまな部分にある複数のファイルに基づいて共有キャッシュを管理する必要がある場合に使用できます。

#### 連結 `package-lock` ファイルの作成と構築

1) カスタム コマンドを設定ファイルに追加します。
{% raw %}```yaml
    commands:
        create_concatenated_package_lock:
        description: "lerna.js で認識されるすべての package-lock.json ファイルを単一のファイルに連結します。 ファイルは、チェックサム ソースとしてキャッシュ キーの一部に使用します"
    parameters:
      filename:
        type: string
    steps:

      - run:
          name: package-lock.json ファイルの単一ファイルへの統合
          command: npx lerna list -p -a | awk -F packages '{printf "\"packages%s/package-lock.json\" ", $2}' | xargs cat > << parameters.filename >>
```
{% endraw %}

2) ビルド時にカスタム コマンドを使用して、連結 `package-lock` ファイルを生成します。
{% raw %}```yaml
    steps:

        - checkout
        - create_concatenated_package_lock:
          filename: combined-package-lock.txt
    ## combined-package-lock.text をキャッシュ キーに使用します
        - restore_cache:
          keys:
            - v3-deps-{{ checksum "package-lock.json" }}-{{ checksum "combined-package-lock.txt" }}
            - v3-deps
```
{% endraw %}

## キャッシュの管理

### キャッシュの有効期限

{:.no_toc} Caches created via the `save_cache` step are stored for up to 15 days.

### キャッシュのクリア
{:.no_toc}
言語または依存関係管理ツールのバージョンが変更され、キャッシュをクリアする必要がある場合は、上の例のような命名戦略を使用し、`config.yml` ファイルのキャッシュ キー名を変更して、変更をコミットします。

<div class="alert alert-info" role="alert">
<b>ヒント:</b> キャッシュは変更不可なので、すべてのキャッシュ キーの先頭にプレフィックスとしてバージョン名 (<code class="highlighter-rouge">v1-...</code>など) を付加すると便利です。 こうすれば、プレフィックスのバージョン番号を増やすだけで、キャッシュ全体を再生成できます。
</div>

たとえば、以下のような場合に、キャッシュ キー名の数字を増やすことでキャッシュをクリアできます。

- npm のバージョンが 4 から 5 に変更されるなど、依存関係管理ツールのバージョンが変更された場合
- Ruby のバージョンが 2.3 から 2.4 に変更されるなど、言語のバージョンが変更された場合
- プロジェクトから依存関係が削除された場合

<div class="alert alert-info" role="alert">
  <b>ヒント:</b> キャッシュ キーに <code class="highlighter-rouge">:、?、&、=、/、#</code> などの特殊文字や予約文字を使用すると、ビルドの際に問題が発生する可能性があるため、注意が必要です。 一般に、キャッシュ キーのプレフィックスには [a-z][A-Z] の範囲の文字を使用してください。
</div>

### キャッシュ サイズ

{:.no_toc} キャッシュ サイズは 500 MB 未満に抑えることをお勧めします。 これは、破損チェックを効率的に実行するための上限のサイズです。500 MB を超えると、チェック時間が非常に長くなります。 キャッシュ サイズは、CircleCI の [Jobs (ジョブ)] ページの `restore_cache` ステップで確認できます。 キャッシュ サイズを増やすこともできますが、キャッシュの復元中に問題が発生したり、ダウンロード中に破損する可能性が高くなるため、お勧めできません。 キャッシュ サイズを抑えるため、複数のキャッシュに分割することを検討してください。

## 基本的な依存関係キャッシュの例

CircleCI 2.0 の手動で構成可能な依存関係キャッシュを最大限に活用するには、キャッシュの対象と方法を明確にする必要があります。 その他の例については、「CircleCI を設定する」の「[save_cache]({{ site.baseurl }}/2.0/configuration-reference/#save_cache)」セクションを参照してください。

ファイルやディレクトリのキャッシュを保存するには、`.circleci/config.yml` ファイルでジョブに `save_cache` ステップを追加します。

```yaml
    steps:
      - save_cache:
          key: my-cache
          paths:
            - my-file.txt
            - my-project/my-dependencies-directory
```

ディレクトリのパスは、ジョブの `working_directory` からの相対パスです。 必要に応じて、絶対パスも指定できます。

**メモ:** 特別なステップ [`persist_to_workspace`]({{ site.baseurl }}/2.0/configuration-reference/#persist_to_workspace) とは異なり、`save_cache` および `restore_cache` は `paths` キーのグロブをサポートしていません。

## キーとテンプレートの使用

各キャッシュ キーは、1 つのデータ キャッシュに対応する*ユーザー定義*の文字列です。 **動的な値**を挿入してキャッシュ キーを作成することができます。これは**テンプレート**と呼ばれます。 キャッシュ キー内の中かっこで囲まれている部分がテンプレートです。 以下を例に考えてみましょう。

```sh
{% raw %}myapp-{{ checksum "package-lock.json" }}{% endraw %}
```

上の例の出力は、このキーを表す一意の文字列です。 ここでは、[チェックサム](https://ja.wikipedia.org/wiki/チェックサム)を使用して、`package-lock.json` の内容を表す一意の文字列を作成しています。

この例では、以下のような文字列が出力されます。

```sh
{% raw %}myapp-+KlBebDceJh_zOWQIAJDLEkdkKoeldAldkaKiallQ<etc>{% endraw %}
```

`package-lock` ファイルの内容が変更された場合、`checksum` 関数は別の一意の文字列を返し、キャッシュを無効化する必要があることが示されます。

キャッシュの `key` に使用するテンプレートを選択するうえでは、キャッシュの保存にはコストがかかること、キャッシュを CircleCI ストレージにアップロードするにはある程度の時間がかかることに留意してください。 ビルドのたびに新しいキャッシュが生成されないように、実際に何かが変更された場合にのみ新しいキャッシュが生成されるような `key` にします。

最初に、プロジェクトの何らかの側面を表す値を含むキーを使用して、キャッシュを保存または復元するタイミングを指定します。 たとえば、ビルド番号が増えたとき、リビジョン番号が増えたとき、依存関係マニフェスト ファイルのハッシュが変更されたときなどです。

以下に、さまざまな目的を持つキャッシュ戦略の例を示します。

-
{% raw %}`myapp-{{ checksum "package-lock.json" }}`

{% endraw %}

- `package-lock.json` ファイルの内容が変わるたびにキャッシュが再生成されます。このプロジェクトのさまざまなブランチで同じキャッシュ キーが生成されます。
-
{% raw %}`myapp-{{ .Branch }}-{{ checksum "package-lock.json" }}
        `
{% endraw %}

- `package-lock.json` ファイルの内容が変更されるたびにキャッシュが再生成されます。このプロジェクトのブランチでそれぞれ異なるキャッシュ キーが生成されます。
-
{% raw %}`myapp-{{ epoch }}`

{% endraw %}

- ビルドのたびに異なるキャッシュ キーが生成されます。

ステップの実行中に、上記のテンプレートが実行時の値に置き換えられ、その置換後の文字列が `key` として使用されます。 以下の表に、使用可能なキャッシュの `key` テンプレートを示します。

| テンプレート                                                 | 説明                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| {% raw %}`{{ checksum "filename" }}`{% endraw %}       | filename で指定したファイルの内容の SHA256 ハッシュを Base64 エンコードした値。ファイルが変更されると、新しいキャッシュ キーが生成されます。 リポジトリにコミットされるファイルのみを指定できます。 依存関係マニフェスト ファイル (`package-lock.json`、`pom.xml`、`project.clj` など) の使用を検討してください。 また、`restore_cache` から `save_cache` までの間にファイルの内容が変更されないようにすることが重要です。ファイルの内容が変更された場合、`restore_cache` のタイミングで使用されるファイルとは異なるキャッシュ キーの下でキャッシュが保存されます。 |
| {% raw %}`{{ .Branch }}`{% endraw %}                   | 現在ビルド中の VCS ブランチ。                                                                                                                                                                                                                                                                                                                               |
| {% raw %}`{{ .BuildNum }}`{% endraw %}                 | このビルドの CircleCI ジョブ番号。                                                                                                                                                                                                                                                                                                                          |
| {% raw %}`{{ .Revision }}`{% endraw %}                 | 現在ビルド中の VCS リビジョン。                                                                                                                                                                                                                                                                                                                              |
| {% raw %}`{{ .Environment.variableName }}`{% endraw %} | 環境変数 `variableName` ([CircleCI からエクスポートされる環境変数](https://circleci.com/ja/docs/2.0/env-vars/#circleci-environment-variable-descriptions)、または特定の[コンテキスト](https://circleci.com/ja/docs/2.0/contexts)に追加した環境変数がサポートされ、任意の環境変数は使用できません)。                                                                                                              |
| {% raw %}`{{ epoch }}`{% endraw %}                     | 協定世界時 (UTC) 1970 年 1 月 1 日午前 0 時 0 分 0 秒からの経過秒数。POSIX や Unix エポックとも呼ばれます。 このキャッシュ キーは、実行のたびに新しいキャッシュを保存する必要がある場合に便利です。                                                                                                                                                                                                                          |
| {% raw %}`{{ arch }}`{% endraw %}                      | OS と CPU (アーキテクチャ、ファミリ、モデル) の情報を取得します。 OS や CPU アーキテクチャに依存するコンパイル済みバイナリをキャッシュする場合に便利です (`darwin-amd64-6_58`、`linux-amd64-6_62` など)。 See [サポートされている CPU アーキテクチャ]({{ site.baseurl }}/2.0/faq/#circleci-ではどの-cpu-アーキテクチャをサポートしていますか)を参照してください。                                                                                                     |
{: class="table table-striped"}

### キーとテンプレートの使用に関する補足説明
{:.no_toc}

- キャッシュに一意の識別子を定義するときは、
{% raw %}`{{ epoch }}`{% endraw %} などの特定度の高いテンプレート キーを過度に使用しないように注意してください。
{% raw %}`{{ .Branch }}`{% endraw %} や
{% raw %}`{{ checksum "filename" }}`{% endraw %} などの特定度の低いテンプレート キーを使用すると、キャッシュが使用される可能性が高くなります。 
- キャッシュ変数には、ビルドで使用している[パラメーター]({{site.baseurl}}/2.0/reusing-config/#executor-でのパラメーターの使用)も使用できます。たとえば、
{% raw %}`v1-deps-<< parameters.varname >>`{% endraw %} のように指定します。
- キャッシュ キーに動的なテンプレートを使用する必要はありません。 静的な文字列を使用し、その名前を「バンプ」(変更) することで、キャッシュを強制的に無効化できます。

### キャッシュの保存および復元の例
{:.no_toc}
以下に、`.circleci/config.yml` ファイルで `restore_cache` と `save_cache` をテンプレートとキーと共に使用する例を示します。

{% raw %}

```yaml
    docker:
      - image: customimage/ruby:2.3-node-phantomjs-0.0.1
        environment:
          RAILS_ENV: test
          RACK_ENV: test
      - image: circleci/mysql:5.6

    steps:

      - checkout
      - run: cp config/{database_circleci,database}.yml

      # Bundler を実行します
      # 可能な場合は、インストールされている gem をキャッシュから読み込み、バンドル インストール後にキャッシュを保存します
      # 複数のキャッシュを使用して、キャッシュ ヒットの確率を上げます

      - restore_cache:
          keys:
            - gem-cache-v1-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
            - gem-cache-v1-{{ arch }}-{{ .Branch }}
            - gem-cache-v1

      - run: bundle install --path vendor/bundle

      - save_cache:
          key: gem-cache-v1-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle

      - run: bundle exec rubocop
      - run: bundle exec rake db:create db:schema:load --trace
      - run: bundle exec rake factory_girl:lint

      # アセットのプリコンパイルを行います
      # 可能な場合は、アセットをキャッシュから読み込み、アセットのプリコンパイル後にキャッシュを保存します
      # 複数のキャッシュを使用して、キャッシュ ヒットの確率を上げます

      - restore_cache:
          keys:
            - asset-cache-v1-{{ arch }}-{{ .Branch }}-{{ .Environment.CIRCLE_SHA1 }}
            - asset-cache-v1-{{ arch }}-{{ .Branch }}
            - asset-cache-v1

      - run: bundle exec rake assets:precompile

      - save_cache:
          key: asset-cache-v1-{{ arch }}-{{ .Branch }}-{{ .Environment.CIRCLE_SHA1 }}
          paths:
            - public/assets
            - tmp/cache/assets/sprockets

      - run: bundle exec rspec
      - run: bundle exec cucumber
```

{% endraw %}

### 部分的な依存関係キャッシュの使用方法
{:.no_toc}

依存関係管理ツールの中には、部分的に復元された依存関係ツリー上へのインストールを正しく処理できないものがあります。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        - gem-cache-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
        - gem-cache-{{ arch }}-{{ .Branch }}
        - gem-cache
```

{% endraw %}

上の例では、2 番目または 3 番目のキャッシュ キーによって依存関係ツリーが部分的に復元された場合に、依存関係管理ツールによっては古い依存関係ツリーの上に誤ってインストールを行ってしまいます。

カスケード フォールバックの代わりに、以下のように単一バージョンのプレフィックスが付いたキャッシュ キーを使用することで、動作の信頼性が高まります。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        - v1-gem-cache-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
```

{% endraw %}

キャッシュは変更不可なので、この方法でバージョン番号を増やすことで、すべてのキャッシュを再生成できます。 この方法は、以下のような場合に便利です。

- `npm` などの依存関係管理ツールのバージョンを変更した場合
- Ruby などの言語のバージョンを変更した場合
- プロジェクトに依存関係を追加または削除した場合

部分的な依存関係キャッシュの信頼性は、依存関係管理ツールに依存します。 以下に、一般的な依存関係管理ツールについて、推奨される部分キャッシュの使用方法をその理由と共に示します。

#### Bundler (Ruby)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。ただし、注意点があります。

Bundler では、明示的に指定されないシステム gem が使用されるため、確定的でなく、部分キャッシュ リストアの信頼性が低下することがあります。

この問題を解決するには、キャッシュから依存関係を復元する前に Bundler をクリーンアップするステップを追加します。

{% raw %}

```yaml
steps:
  - run: bundle clean --force
  - restore_cache:
      keys:
        # when lock file changes, use increasingly general patterns to restore cache
        - v1-gem-cache-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
        - v1-gem-cache-{{ arch }}-{{ .Branch }}-
        - v1-gem-cache-{{ arch }}-
  - run: bundle install
  - save_cache:
      paths:
        - ~/.bundle
      key: v1-gem-cache-{{ arch }}-{{ .Branch }}-{{ checksum "Gemfile.lock" }}
```

{% endraw %}

#### Gradle (Java)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。

Gradle リポジトリは、規模が大きく、一元化や共有が行われることが想定されています。 生成されたアーティファクトのクラスパスに実際に追加されるライブラリに影響を与えることなく、キャッシュの一部を復元できます。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        # ロック ファイルが変更されたら、パターンが一致する範囲を少しずつ広げてキャッシュを復元します
        - gradle-repo-v1-{{ .Branch }}-{{ checksum "dependencies.lockfile" }}
        - gradle-repo-v1-{{ .Branch }}-
        - gradle-repo-v1-
  - save_cache:
      paths:
        - ~/.gradle
      key: gradle-repo-v1-{{ .Branch }}-{{ checksum "dependencies.lockfile" }}
```

{% endraw %}

#### Maven (Java) および Leiningen (Clojure)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。

Maven リポジトリは、規模が大きく、一元化や共有が行われることが想定されています。 生成されたアーティファクトのクラスパスに実際に追加されるライブラリに影響を与えることなく、キャッシュの一部を復元できます。

Leiningen も内部で Maven を利用しているため、キャッシュの一部を復元できます。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        # ロック ファイルが変更されたら、パターンが一致する範囲を少しずつ広げてキャッシュを復元します
        - maven-repo-v1-{{ .Branch }}-{{ checksum "pom.xml" }}
        - maven-repo-v1-{{ .Branch }}-
        - maven-repo-v1-
  - save_cache:
      paths:
        - ~/.m2
      key: maven-repo-v1-{{ .Branch }}-{{ checksum "pom.xml" }}
```

{% endraw %}

#### npm (Node)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。ただし、npm5 以降を使用する必要があります。

npm5 以降でロック ファイルを使用すると、部分キャッシュ リストアを安全に行うことができます。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        # ロック ファイルが変更されたら、パターンが一致する範囲を少しずつ広げてキャッシュを復元します
        - node-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}
- node-v1-{{ .Branch }}-
        - node-v1-
  - save_cache:
      paths:
        - ~/usr/local/lib/node_modules  # 場所は npm のバージョンによって異なります
      key: node-v1-{{ .Branch }}-{{ checksum "package-lock.json" }}
```

{% endraw %}

#### pip (Python)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。ただし、Pipenv を使用する必要があります。

Pip では、`requirements.txt` で明示的に指定されていないファイルを使用できます。 [Pipenv](https://docs.pipenv.org/) を使用するには、ロック ファイルでバージョンを明示的に指定する必要があります。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        # ロック ファイルが変更されたら、パターンが一致する範囲を少しずつ広げてキャッシュを復元します
        - pip-packages-v1-{{ .Branch }}-{{ checksum "Pipfile.lock" }}
        - pip-packages-v1-{{ .Branch }}-
        - pip-packages-v1-
  - save_cache:
      paths:
        - ~/.local/share/virtualenvs/venv  # このパスは、pipenv が virtualenv を作成する場所によって異なります
      key: pip-packages-v1-{{ .Branch }}-{{ checksum "Pipfile.lock" }}
```

{% endraw %}

#### Yarn (Node)
{:.no_toc}

**部分キャッシュ リストアを使用しても安全でしょうか?** はい。

Yarn では、部分キャッシュ リストアを実行するために、既にロック ファイルが使用されています。

{% raw %}

```yaml
steps:
  - restore_cache:
      keys:
        # ロック ファイルが変更されたら、パターンが一致する範囲を少しずつ広げてキャッシュを復元します
        - yarn-packages-v1-{{ .Branch }}-{{ checksum "yarn.lock" }}
        - yarn-packages-v1-{{ .Branch }}-
        - yarn-packages-v1-
  - save_cache:
      paths:
        - ~/.cache/yarn
      key: yarn-packages-v1-{{ .Branch }}-{{ checksum "yarn.lock" }}
```

We recommend using `yarn --frozen-lockfile --cache-folder ~/.cache/yarn` for two reasons.

1) `--frozen-lockfile` ensures that a whole new lockfile is created and it also ensures your lockfile isn't altered. This allows for the checksum to stay relevant and your dependencies should identically match what you use in development.

2) The default cache location depends on OS. `--cache-folder ~.cache/yarn` ensures we're explitly matching our cache save location.

{% endraw %}

## キャッシュ戦略のトレードオフ

In cases where the build tools for your language include elegant handling of dependencies, partial cache restores may be preferable to zero cache restores for performance reasons. If you get a zero cache restore, you have to reinstall all of your dependencies, which can result in reduced performance. One alternative is to get a large percentage of your dependencies from an older cache instead of starting from zero.

However, for other types of languages, partial caches carry the risk of creating code dependencies that are not aligned with your declared dependencies and do not break until you run a build without a cache. If the dependencies change infrequently, consider listing the zero cache restore key first.

Then, track the costs over time. If the performance costs of zero cache restores (also referred to as a *cache miss*) prove to be significant over time, only then consider adding a partial cache restore key.

Listing multiple keys for restoring a cache increases the odds of a partial cache hit. However, broadening your `restore_cache` scope to a wider history increases the risk of confusing failures. For example, if you have dependencies for Node v6 on an upgrade branch, but your other branches are still on Node v5, a `restore_cache` step that searches other branches might restore incompatible dependencies.

### ロック ファイルの使用
{:.no_toc}

Language dependency manager lockfiles (for example, `Gemfile.lock` or `yarn.lock`) checksums may be a useful cache key.

An alternative is to do `ls -laR your-deps-dir > deps_checksum` and reference it with

{% raw %}

`{{ checksum "deps_checksum" }}`{% endraw %}. For example, in Python, to get a more specific cache than the checksum of your `requirements.txt` file you could install the dependencies within a virtualenv in the project root `venv` and then do `ls -laR venv > python_deps_checksum`.

### 言語ごとに異なるキャッシュを使用する
{:.no_toc}

It is also possible to lower the cost of a cache miss by splitting your job across multiple caches. By specifying multiple `restore_cache` steps with different keys, each cache is reduced in size thereby reducing the performance impact of a cache miss. Consider splitting caches by language type (npm, pip, or bundler) if you know how each dependency manager stores its files, how it upgrades, and how it checks dependencies.

### 高コストのステップのキャッシュ
{:.no_toc}

Certain languages and frameworks have more expensive steps that can and should be cached. Scala and Elixir are two examples where caching the compilation steps will be especially effective. Rails developers, too, would notice a performance boost from caching frontend assets.

Do not cache everything, but *do* consider caching for costly steps like compilation.

## ソースのキャッシュ

As in CircleCI 1.0, it is possible and oftentimes beneficial to cache your git repository, thus saving time in your `checkout` step—especially for larger projects. Here is an example of source caching:

{% raw %}

```yaml
    steps:
      - restore_cache:
          keys:
            - source-v1-{{ .Branch }}-{{ .Revision }}
            - source-v1-{{ .Branch }}-
            - source-v1-

      - checkout

      - save_cache:
          key: source-v1-{{ .Branch }}-{{ .Revision }}
          paths:
            - ".git"
```

{% endraw %}

In this example, `restore_cache` looks for a cache hit from the current git revision, then for a hit from the current branch, and finally for any cache hit, regardless of branch or revision. When CircleCI encounters a list of `keys`, the cache will be restored from the first match. If there are multiple matches, the most recently generated cache will be used.

If your source code changes frequently, we recommend using fewer, more specific keys. This produces a more granular source cache that will update more often as the current branch and git revision change.

Even with the narrowest `restore_cache` option ({% raw %}`source-v1-{{ .Branch }}-{{ .Revision }}`{% endraw %}), source caching can be greatly beneficial when, for example, running repeated builds against the same git revision (i.e., with [API-triggered builds](https://circleci.com/docs/api/v1/#trigger-a-new-build-by-project-preview)) or when using Workflows, where you might otherwise need to `checkout` the same repository once per Workflows job.

That said, it's worth comparing build times with and without source caching; `git clone` is often faster than `restore_cache`.

**NOTE**: The built-in `checkout` command disables git's automatic garbage collection. You might choose to manually run `git gc` in a `run` step prior to running `save_cache` to reduce the size of the saved cache.

## 関連項目
{:.no_toc}

[Optimizations]({{ site.baseurl }}/2.0/optimizations/)