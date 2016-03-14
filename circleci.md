## File structure and content

以下の六つの primary section より作られます。それぞれの section はテスト実行フェーズを表します。

- machine: VM をあなたの選択や要件に合わせます
- checkout: あなたの git repo を clone し checkout します
- dependencies: あなたのプロジェクトの言語固有の要件をセットアップします
- database: あなたのテストのためのデータベースを用意します
- test: あなたのテストを実行します
- deployment: あなたのコードをあなたの Web サーバに deploy します

他に general と experimental というセクションがある模様。

- general: 固有のフェーズに関連しない全般的なビルド関連設定
- experimental: 検討中の設定変更の early access preview

ほとんどのプロジェクトはフェーズの多くのための何らかを記述する必要はないでしょう。

セクションは bash コマンドのリストを含みます。コマンドを記述しない場合でも CircleCI がコードから推論するでしょう。コマンドはファイルに記載された順に実行され、全てのテストコマンドが実行されますが、設定中の非ゼロ値の戻りは早めにビルド失敗を発生させるでしょう。`override`、`pre`、`post` の追加で「どこ」や「いつ」コマンドが動くかを CircleCI が推論するコマンドに合わせて変更することができます。その仕組みは以下です。

- pre: CircleCI が推論するコマンドの前に実行されるコマンド
- override: CircleCI が推論するコマンドの代わりに実行されるコマンド
- post: CircleCI が推論するコマンドの後に実行されるコマンド

それぞれのコマンドは個別の shell にて実行されます。そうした事情で、前のコマンドにおける環境は共有されませんので `export foo=bar` が機能しないことに注意してください。もしグローバルな環境変数を設定したいのであれば、それらを以下に記載する Machine configuration セクションに記述することができます。

### Modifiers

修飾子 (modifier) の追加によりそれぞれのコマンドを微調整できます。使うことができるものは以下。

- timeout: 出力なしで長い時間動いている場合、強制終了します (デフォルトは 600 秒)
- pwd: この値をカレントディレクトリとしてコマンドを実行します
- environment: コマンドのために設定される環境変数のリストを生成する hash 
- parallel: (test セクションのコマンドのみ) [manually set up parallelism](https://circleci.com/docs/parallel-manual-setup) を使う場合はこの値を true にします
- files: file list で識別される files はコマンド引数に追加されます。files はビルドが動いている全てのコンテナにわたって配布されます。詳細は [manual parallelism setup document](https://circleci.com/docs/parallel-manual-setup#auto-splitting) を確認のこと
- background: true の場合、コマンドをバックグラウンドで実行します。shell コマンドの末端に '&' を付けたものに似ていますが ssh 経由で確実に動きます。試験が接続するサーバを開始するのに便利です

YAML が新たなプロパティを追加する度、インデントについて非常に厳密であることに注意してください。このため modifiers はコマンドから 1 レベルインデントされていなければなりません。以下の例では `bundle install` が `timeout`、`environment`、および `pwd` のキーのハッシュ値とともに扱われます。

```
dependencies:
  override:
    - bundle install:
        timeout: 240
        environment:
          foo: bar
          foo2: bar2
        pwd:
          test_dir
```

## Machine configuration

`machine` セクションはテストを実行する VM の設定を可能します。

以下に `machine` セクションで設定される可能性があるものの一例です。

```
machine:
  timezone:
    America/Los_Angeles
  ruby:
    version: 1.9.3-p0-falcon

test:
  post:
    - bundle exec rake custom:test:suite
```

この例では `time zone` の設定と `Ruby version` と patchset の選択、そしてコマンドが終了した後で動作するカスタムテストコマンドが追加されている。

`machine` セクションでは `pre` および `post` がサポートされていますが、`override` はそうではありません。`machine pre` セクションにおいてはカスタム環境変数は有効ではないことに注意してください。

以下はどのように `circle.yml` を `pre` を使って CircleCI に導入されているものと異なるバージョンの `phantomjs` を導入するかという例です。

```
machine:
  pre:
    - curl -k -L -o phantomjs.tar.bz2 http://phantomjs.googlecode.com/files/phantomjs-1.8.2-linux-x86_64.tar.bz2
    - tar -jxf phantomjs.tar.bz2
```

### Environment

`machine` セクションに `environment` の追加により、ビルドにおける全てのコマンドのための環境変数を設定します。CircleCI は全てのコマンドに新たな shell を使うことを覚えておいて下さい。`export foo=bar` は前に書いたとおり動きません。代わりに以下を include する必要があります。

```
machine:
  environment:
    foo: bar
    baz: 123
```

この方法を使わない場合、[a number of other options](https://circleci.com/docs/environment-variables) があります。

### Timezone

マシンの timezone はデフォルトで UTC です。プロダクションサーバと同じ timezone にしたい場合 `timezone` を使います。時刻を開発機の timezone に変更することは自己責任です。

この modifier は /etc/timezone を書き換え、すべての database および動作しているサービスを再起動するよう CircleCI に伝えます。この modifier は IANA の time zone database にリストされているいくつかの timezone をサポートします。そのリストは UNIX マシンの `/usr/share/zoneinfo` であるいは Wikipedia list of TZ database time zones の TZ カラムから見つけることができます。

デベロッパ、特に異なる timezone に跨がって協力開発しているものはプロダクションサーバにおいて UTC を使うということに注意をしてください。この選択で恐しい Daylight Saving Time (DST) バグを回避できます。

### Hosts

IP adress にさまざまな hostname をアサインするために `/etc/hosts` にエントリを追加する必要があるかもしれません。以下の方法でそのマッピングが可能です。

```
machine:
  hosts:
    dev.circleci.com: 127.0.0.1
    foobar: 1.2.3.4
```

CircleCI はこれらの値で `/etc/hosts` を自動で更新するでしょう。hostname は well-formed である必要があります。CircleCI は英数、ハイフン、ドットを含む hostname のみを受け入れるでしょう。

### Ruby version

CircleCI は Ruby のバージョンを管理するために RVM を使います。.rvmrc、.ruby-version または Gemfile に記載された Ruby のバージョンが使われます。これらのファイルが無い場合、1.9.3-p448 あるいは 1.8.7-p358 の適当と思われるバージョンを使います。別の Ruby バージョンを使う場合、`machine` セクションの情報に含めることで CircleCI に知らせてください。以下がその方法の例です。

```
machine:
  ruby:
    version: 1.9.3-p0-falcon
```

サポートしている Ruby のバージョンの完全なリストは[ここ](https://circleci.com/docs/environment#ruby)です。

### Node.js version

CircleCI は Node のバージョン管理に NVM を使っています。完全なリストは [supported Node versions](https://circleci.com/docs/environment#nodejs)にあります。バージョンの記述が無い場合は CircleCI は 0.10.33 を使います。

以下がテストに使われる Node.js のバージョンを設定する方法の例です。

```
machine:
  node:
    version: 0.11.8
```

### Java version

以下がテストに使われる Java のバージョンを設定する方法の例です。

```
machine:
  java:
    version: openjdk7
```

Java のデフォルトのバージョンは `oraclejdk7` です。完全なリストは [supported Java versions](https://circleci.com/docs/environment#java)にあります。

### PHP version

CircleCI は php-build および phpenv を PHP バージョン管理のために使っています。以下がテストに使われる PHP のバージョンを設定する方法の例です。

```
machine:
  php:
    version: 5.4.5
```

完全なリストは [supported PHP versions](https://circleci.com/docs/environment#php)にあります。

### Python version

CircleCI は pyenv を Python バージョン管理のために使っています。以下がテストに使われる Python のバージョンを設定する方法の例です。

```
machine:
  python:
    version: 2.7.5
```

完全なリストは [supported Python versions](https://circleci.com/docs/environment#python) にあります。

### GHC version

`circle.yml` において [number of available GHC versions](https://circleci.com/docs/environment#haskell) から選択可能です。

```
machine:
  ghc:
    version: 7.8.3
```

### Other languages

[test environment](https://circleci.com/docs/environment) ドキュメントに Python、Closure、C/C++、Golang および Erlang を含んだ他の言語に関する設定情報があります。

### Databases and other services

CircleCI は沢山の [databased and other services](https://circleci.com/docs/environment#databases) をサポートします。最も人気のある Postgres、MySQL、Redis そして MongoDB を含む (localhost にバインドされた) ものが我々のビルドマシンデフォルトで動いています。

他のデータベースやサービスは `services` セクションから enable にできます。

```
machine:
  services:
    - cassandra
    - elasticsearch
    - rabbitmq-server
    - riak
    - beanstalkd
    - couchbase-server
    - neo4j
    - sphinxsearch
```

## Code checkout from Github

`checkout` セクションはたいていかなり素の状態です (普通は何もしない) が、セクションに配置する必要があるかもしれない一般的な例を列挙します。checkout フェーズ後まで circle.yml は読まないため、ここでは `post` のみがサポートされます。

#### Example: git submodule の使用

```
checkout:
  post:
    - git submodule sync
    - git submodule update --init
```

#### Example: CircleCI での設定ファイルの上書き

```
checkout:
  post:
    - mv config/.app.yml config/app.yml
```

## Project-specific dependencies

ほとんどの Web Programming 言語およびフレームワークは Ruby の bundler、Node.js の npm そして Python の pip を含むいくつかの依存関係の記述のフォームを持っており、CircleCI は自動でそれらの依存関係を探すコマンドを実行します。

`dependencies` コマンドを修正するために `override`、`pre` および `post` を使うことができます。以下に `dependencies` セクションの例を列挙します。

#### Example: npm と Node.js

```
dependencies:
  override:
    - npm install
```

#### Example: bundler の特定のバージョンを使う

```
dependencies:
  pre:
    - gem uninstall bundler
    - gem install bundler --pre
```

### Bundler flags

プロジェクトが bundler を含む場合、bundle install から除外される依存グループのリストするために `without` を含むことができます。

```
dependencies:
  bundler:
    without: [production, osx]
```

### Custom Cache Directories

CircleCI はビルド間の依存関係を cache します。cache にカスタムなディレクトリを含めるにはビルド間で cache したいディレクトリをリストする `cache_directories` を使うことができます。以下は二点のカスタムディレクトリを cache するための例です。

```
dependencies:
  cache_directories:
    - "assets/cache"    # relative to the build directory
    - "~/assets/output" # relative to the user's home directory
```

cache は dependency の後に起こり cache_directories に記載されたディレクトリはその前に有効になっている必要があります。

cache はプライベートであり、他のプロジェクトと共有されません。

## Database setup

Web フレームワークは一般的にデータベースの生成、schema の導入、そして migration のコマンドを含んでいます。データベースを modify するために `override`、`pre` および `post` を使うことができます。詳細は [Setting up your test database](https://circleci.com/docs/manually#databases) を確認してください。

わたしたちが推測した `database.yml` がイケてない場合には、わたしたちの設定コマンド (以下に例示) を `override` しなければならないかもしれません。そのような場合にはわたちたちに連絡し、わたしたちの推論を向上させるために Circle にお知らせください。

```
database:
  override:
    - mv config/database.ci.yml config/database.yml
    - bundle exec rake db:create db:schema:load --trace
```

FYI: `machine` セクションの `environment` modifier によるデータベース設定の格納場所を指定するオプションがあります。

```
machine:
  environment:
    DATABASE_URL: postgres://ubuntu:@127.0.0.1:5432/circle_test
```

## Running your tests

テストの一番重要なパートはテストの実行です。

CircleCI は`overrice`、`per`、および `post` の `test` セクションにおける使用をサポートしています。しかし、このセクションは一つの小さな違いがあります。テストコマンドは一つが失敗しても全てが動きます。最初のエラーだけでなく全てのテストの失敗をあなたに教えることができるのです。

#### Example: Rspec の後に spinach を動かす

```
test:
  post:
    - bundle exec rake spinach:
        environment:
          RAILS_ENV: test
```

#### Example: 特別なディレクトリで phpunit を動かす

```
test:
  override:
    - phpunit my/special/subdirectory/tests
```

CircleCI もテスト中に使われる file glob をリストできる `minitest_globs` の使用をサポートします。

並行テスト時のデフォルト設定で CircleCI は test/unit、test/integration、test/functional ディレクトリの全てのテストを実行します。あなたはあなた自身で標準のディレクトリを置換するために minitest_globs を追加できます。これはあなたが追加の又は非標準ディレクトリを持ち、MiniTest を並列でテストする時にのみ必要とされます

#### Example: minitest_globs

```
test:
  minitest_globs:
    - test/integration/**/*.rb
    - test/extra-dir/**/*.rb
```

## Deployment

`deployment` セクションはオプションです。あなたはステージングあるいは本番環境にデプロイするためのコマンドを実行することができます。これらのコマンドはビルドが成功した後のみ実行されます。

```
deployment:
  production:
    branch: production
    commands:
      - ./deploy_prod.sh
  staging:
    branch: master
    commands:
      - ./deploy_staging.sh
```

`deployment` セクションは複数のサブセクションから成ります。以下の例では片方は `production`、もう片方は `staging` という二つがあります。サブセクションの名前はユニークでなければなりません。それぞれのサブセクションは複数の branch をリストすることができますが、少なくともこれらのフィールドの一つは `branch` の名前が付けられている必要があります。複数 branch のインスタンスでは、ビルドされる branch の最初のものが実行されているものです。以下の例では開発者がリストされている三つの branch のいずれかを push した場合、`merge_to_master.sh` というスクリプトが実行されます。

```
deployment:
  automerge:
    branch: [dev_alice, dev_bob, dev_carol]
    commands:
      - ./merge_to_master.sh
```

`branch` フィールドも '/' で囲まれた正規表現で記述することができます (例: /feature_.*/)。

```
deployment:
  feature:
    branch: /feature_.*/
    commands:
      - ./deploy_feature.sh
```

必要に応じて deployment サブセクションにリポジトリの `owner` を記載することもできます。これはプロジェクトの複数 fork がある場合に有用ですが、片方のみがデプロイされる必要があります。例えば、プロジェクトガ "circleci" に依存しており、このような deployment サブセクションは deploy のみされるでしょう。そして他のユーザは deploy をトリガしない彼らの fork の master ブランチを push することができます。

```
deployment:
  master:
    branch: master
    owner: circleci
    commands:
      - ./deploy_master.sh
```

### Tags

branch に基づいて deploy することに加えてタグに基づいた deploy ができます。

通常、タグの push はビルドを動かしません。あなたが作成したタグの名前にマッチする tag プロパティがある deployment 設定がある場合、マッチする deployment セクションとビルドが実行されるでしょう。

[Cutting a release on Github](https://help.github.com/articles/creating-releases/) はタグを生成し、同じルールに従います。

注：現在、注釈付きのタグのみがサポートされています。lightweight タグの push はビルドをトリガしません。

以下の例では release-v1.05 という名前のタグを push がビルドと deploy をトリガします。qa-9502 タグの push はビルドをトリガしません。

```
deployment:
  release:
    tag: /release-.*/
    owner: circleci
    commands:
      - ./deploy_master.sh
```

`branch` プロパティと同様に、`tag` プロパティは正確な文字列又は正規表現が指定できます。

一般的な慣習は、1.2.3 というバージョンのために `v1.2.3` というタグを作ることです。以下の正規表現はそのパターンを実装するでしょう。

```
/v[0-9]+(\.[0-9]+)*/
```

v1、v1.2 そして v1.2.3 (等々) がマッチします。

### SSH keys

SSH アクセスが要求されるサーバに deploy する場合、CircleCI に鍵をアップロードする必要があります。CircleCI の UI はプロジェクトの Project Settings -> SSH keys というページでこれを行なうことが可能です。一つ以上の SSH key の追加およびサブミットがあなたのマシンに dploy するために必要です。Hostname フィールドをブランクのままにしておくことで、鍵は全てのホスト向けに使われるでしょう。

### Heroku

CircleCI も Heroku への deploy にファーストクラスなサポートを用意しています。`appname` に `git push` したいアプリを指定しなさい。ビルド成功すると、自動で deploy します。

```
deployment:
  staging:
    branch: master
    heroku:
      appname: foo-bar-123
```

Heroku に deploy する設定は一つの余分なステップを要求します。Heroku のアーキテクハとセキュリティモデルのため、わたしたちは特定のユーザとして deploy する必要があります。あなたのプロジェクトのメンバ (おそらくあなた) はそのユーザとして登録を行う必要があります。CircleCI の UI はプロジェクトの Project Setting > Heroku settings ページでそれをすることができます。

### Heroku with pre or post-deployment steps

Heroku に deploy し、その前後でコマンドを実行したい場合、'normal' な deployment な文法を使わなければなりません。

```
deployment:
    production:
      branch: production
      commands:
        - git push git@heroku.com:foo-bar-123.git $CIRCLE_SHA1:master
        - heroku run rake db:migrate --app foo-bar-123
```

## Notifications

CircleCI は email による個別の通知を送付します。

ユーザ毎の email に加えて CircleCI はプロジェクトごとの通知を送付します。CircleCI はビルド完了時の webhook 送付もサポートします。CircleCI も HipChat、Campfire、Flowdock や IRC 通知をサポートします。プロジェクトの Project Settings > Notification ページでこれらの通知を設定します。

この例では指定された URL への JSON パケットを POST します。

```
notify:
  webhooks:
    # A list of hook hashes, containing the URL field
    - url: https://example.com/hooks/circle
```

JSON パケットは "payload" というキーで wrap されている事以外は同じビルドのための Build API 呼び出しの結果と同一です。

```
{
  "payload": {
    "vcs_url" : "https://github.com/circleci/mongofinil",
    "build_url" : "https://circleci.com/gh/circleci/mongofinil/22",
    "build_num" : 22,
    "branch" : "master",
    ...
  }
}
```

per branch build notification セクションにおいてビルド通知をチャットのチャネルで取得したいブランチのブラックリストまたはホワイトリストの記述を設定できる実験的な設定もあります。

## Specifying branches to build

CircleCI はデフォルトでリポジトリの任意のブランチの全ての push を試験します。全てのブランチの試験は全てのブランチでの品質を維持し、ブランチがデフォルトブランチにマージされる時の信頼を担保します。

しかし、CircleCI でビルドが blacklist ブランチかもしれません。この例は circle でのビルドから gh-pages を除外します。

```
general:
  branches:
    ignore:
      - gh-pages # list of branches to ignore
      - /release\/.*/ # or ignore regexes
```

ホワイトリストブランチだけビルドをトリガします。この例は circle での `master` および `feature-*` ブランチのビルドを制限します。

```
general:
  branches:
    only:
      - master # list of branches to build
      - /feature-.*/ # or regexes
```

わたしたちは branch をホワイトリスト化することは推奨しません。それは作業注のコードが長い間統合されず試験されないことを意味し、わたしたちは試験されないコードがマージされる時の問題につながることを確認しています。

`circle.yml` はブランチ毎の設定ファイルです、そしてあるブランチを無視するブランチはそのブランチにのみ影響するでしょう。

## Specifying custom artivacts directories and files

Circle はデフォルトでリポジトリ root の全てのコマンドを実行します。しかし root の代わりにサブディレクトリにアプリケーションコードを格納した場合、`circle.yml` にビルドディレクトリを記述できます。例えば、api サブディレクトリをビルドディレクトリに設定するために、以下の設定を使うことができます。

```
general:
  build_dir: api
```

Circle はそのディレクトリのビルドコマンドと同様に推論を実行します。

### カスタムアーティファクトディレクトリとファイルの指定

(デフォルトの $CIRCLE_ARTIFACTS ディレクトリに加えて) artifact として保存されるファイルやディレクトリの指定ができます。

```
general:
  artifacts:
    - "selenium/screenshots" # relative to the build directory
    - "~/simplecov" # relative to the user's home directory
    - "test.txt" # a single file, relative to the build directory
```

ワイルドカードのようなより複雑な artifact の取り扱いが要求する場合、$CIRCLE_ARTIFACTS ディレクトリへの成果物を移動することを推奨します。

```
test:
  post:
    - mkdir $CIRCLE_ARTIFACTS/json_output
    - mv solo/target/*.json $CIRCLE_ARTIFACTS/json_output
```

## Experimental configuration

わたしたちの実験的なセクションは追加を検討している新しい設定オプションのアーリープレビューを提供する方法です。これらの設定は予告なしに変更する傾向があります。

### Per branch build notification in chat channels

現時点で有効な唯一の実験的な設定はブランチの名前に基いたチャットチャネルへのビルド通知のためのブラックリスト、ホワイトリストメカニズムです。

"ignore" および "only" 設定の動作は Branches セクションにおけるビルドのためのブランチのブラックリストおよびホワイトリストと同様です。それぞれの設定は文字列ないし正規表現のリストを取ります。正規表現は '/' でその値が囲まれます。

以下の設定は "dev" または "experiment" で始まるか、'sandbox' という名前のブランチのビルドのためのチャットチャネル宛てビルド通知を抑制します

```
experimental:
  notify:
    branches:
      ignore:
        - /dev.*/
        - /experiment.*/
        - sandbox
```

代わりに、ホワイトリストにマッチするブランチのための通知の送付が可能です。以下の設定は master ブランチおよび "feature" で始まるブランチの通知のみ送付するでしょう。

```
experimental:
  notify:
    branches:
      only:
        - master
        - /feature-.*/
```

それらを組み合わせることができます。その場合ホワイトリストにマッチし、ブラックリストにマッチしないブランチの名前のみ、通知を受けとります。

```
experimental:
  notify:
    branches:
      only:
        - /feature.*/
      ignore:
        - /feature\.experiment.*/
```

"feature-1" という名前のブランチは通知が送付されますが、"feature.experiment-1' は送付されないでしょう。
