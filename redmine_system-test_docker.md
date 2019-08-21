# DockerでRedmineのSystem Testを動かしてみた

業務でRedmineのPlugin開発をしているのですが、
せっかくDocker上で開発しているので、System TestもDocker上で実行できないかと思い試行錯誤してみました。

## Dockerで環境を整える

まず、ディレクトリの構成です。

```sh:ディレクトリ構成
workspace/
 ├── Dockerfile
 ├── docker-compose.yml
 ├── docker-compose-dev.yml
 ├── docker-sync.yml
 ├── docker-compose-test.yml
 ├── entrypoint.sh
 ├── redmine/
 └── postgers/
```

`redmine`は[ここ](https://github.com/redmine/redmine)からclone。
`postgres`ディレクトリは、`docker-compose`でコンテナを起動すれば勝手に作成されるはず。
`postgres`以下にはPostgreSQLのデータが保存されます。

各ファイルの内容です。

### `Dockerfile`

基本の`Dockerfile`。

今回は`ruby:2.5`のイメージを使用します。

```Dockerfile:Dockerfile
FROM ruby:2.5

RUN gem install bundler
RUN mkdir /redmine
WORKDIR /redmine
ENV BUNDLE_PATH /redmine/vendor/bundle
ENV RAILS_ENV development

COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]1
```

`redmine`は後からvolumeするのでイメージの中ではディレクトリを作成するだけです。

実際はRedmineもイメージに追加して、`files`、`plugins`だけvolumeする方がいいかもしれません。
開発用の環境なので、環境変数は`ENV RAILS_ENV development`を指定します。

### `entrypoint.sh`

イメージの作成に使います。
公式のRails用のサンプルを参考にしています。(https://docs.docker.com/compose/rails/)

```entrypoint.sh:sh
#!/bin/bash
set -e

rm -f /redmine/tmp/pids/server.pid

exec "$@"
```

### `docker-compose.yml`

Redmineの開発環境用の`docker-compose.yml`です。

```yml:docker-compose.yml
version: '3'

services:
  redmine:
    build: .
    image: redmine:4
    environment:
      BUNDLE_PATH: /redmine/vendor/bundle
      RAILS_ENV: development
      DB_HOST_NAME: postgres
    volumes:
      - ./redmine:/redmine
    ports:
      - 3000:3000
    depends_on:
      - postgres
  postgres:
    image: postgres:9.6
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

`image:`属性を指定すると、docker-composeでbuildしたイメージの名前を指定できます。
`BUNDLE_PATH`と`RAILS_ENV`は`Dockerfile`で指定しているのでここで指定する必要はありません。

### `docker-compose-dev.yml`

`docker-sync`用の設定です。

System Testはかなり遅いので、`docker-sync`は使用した方がいいと思います。
docker-syncは別にインストールする必要があります。

```yml:docker-compose-dev.yml
version: '3'

services:
  redmine:
    volumes:
      - redmine4_volume:/redmine
  postgres:
    volumes:
      - redmine4_postgres_volume:/var/lib/postgresql/data

volumes:
  redmine4_volume:
    external: true
  redmine4_postgres_volume:
    external: true
```

`servces`には`docker-compose.yml`での設定と変更したい属性と追加したい属性のみを記述します。

`volumes`にはdocker-syncで定義するvolumeの名前を宣言します。

### `docker-sync.yml`

docker-syncの設定ファイルです。

```yml:docker-sync.yml
version: '2'

syncs:
  redmine4_volume:
    src: './redmine'
    sync_strategy: 'unison'
  redmine4_postgres_volume:
    src: './postgresql_data'
    sync_strategy: 'unison'
```

`src`は相対パスで記述してますが、絶対パスで書いた方が安心だと思います。

`docker-sync.yml`は拡張子を`yaml`にしたら起動できませんでした。

### `docker-compose-test.yml`

test環境用のcomposeファイルです。

```yml:docker-compose-test.yml
version: '3'

services:
  redmine:
    environment:
      RAILS_ENV: test
    depends_on:
      - chrome
  selenium-hub:
    image: selenium/hub:3.141.59-selenium
    ports:
      - 4444:4444
  chrome:
    image: selenium/node-chrome-debug:3.141.59-selenium
    shm_size: 2g
    depends_on:
      - selenium-hub
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
    ports:
      - 5900:5900
```

`RAILS_ENV`はテスト環境なので`test`に変更します。

`chrome`はchromeドライバです。
redmineのシステムテストを動かすために必要です。
`selenium-hub`がどうして必要なのか、まだ理解できてないんですが、多分`chrome`を使うために必要なんじゃないかと思ってます。

## Redmineを起動します。

```sh:Terminal
$ cd /path/to/workspace
$ docker-compose build
$ docker-compose run --rm redmine bundle install
$ docker-compose run --rm redmine bundle exec rake db:create db:migrate
```

Redmineの環境が整うまでRedmineの起動ができないので`run`コマンドを使用して環境を整えます。
`--rm`オプションはrunコマンドで作成されたコンテナを削除するためのコマンドです。

ここまででRedmineの準備は完了です！

## System Testを動かす

まずはコンテナ間でネットワークが繋がっているかどうかの確認です。

docker-syncを起動して起きます。

```sh:Terminal
$ docker-sync start
```

### chrome側からRedmineが見えるか確認

chromeとRedmineが繋がっているのか確認したかったので、まずはchromeのコンテナからRedmineにアクセスできるかを確認します。

まずはRedmineを起動。

```
$ docker-compose -f docker-compose.yml -f docker-compose-dev.yml up
```

Redmineを起動したらホストのブラウザを立ち上げて、アドレスバーに`http://localhost:5900`を入力します。
画面共有をするか聞かれるので`OK`を選択。
パスワードを聞かれたら`secret`、黒い画面が立ち上がるはずです。

起動した画面上で*右クリック → Application → Network → chrome*。chromeコンテナ上のchromeが起動できます。

立ち上げたchromeのアドレスバーに`http://redmine:3000`を入力するとRedmineにアクセスできます。
URLの`redmine`は`docker-compose.yml`で指定したサービス名です。
Redmineの画面が表示されれば成功です。一度コンテナを落とします。

```
$ docker-compose down
```

### RedmineのTestファイルを修正

Redmineのシステムテストの設定を、dockerコンテナ上で動くように変更します。

[RailsのAPI](https://api.rubyonrails.org/v5.1.3/classes/ActionDispatch/SystemTestCase.html)とか
`actionpack-5.2.3/lib/action_dispatch/system_testing/driver.rb`と`actionpack-5.2.3/lib/action_dispatch/system_testing/browser.rb`のソースを参考にしながら
`application_system_test_case.rb`の設定を追加してみました。

```ruby:redmine/test/application_system_test_case.rb
  driven_by :selenium, using: :chrome, screen_size: [1024, 900], options: {
      desired_capabilities: Selenium::WebDriver::Remote::Capabilities.chrome(
        'chromeOptions' => {
          'prefs' => {
            'download.default_directory' => DOWNLOADS_PATH,
            'download.prompt_for_download' => false,
            'plugins.plugins_disabled' => ["Chrome PDF Viewer"]
          }
        }
      ),
      browser: :remote,
      url: 'http://selenium-hub:4444/wd/hub'
    }

  setup do
    clear_downloaded_files
    Setting.delete_all
    Setting.clear_cache
    Capybara.app_host = 'http://redmine:3000'
  end
```

`driven_by`の引数のハッシュに、ブラウザの種類`browser: :remote`、
ブラウザのURL`url: 'http://selenium-hub:4444/wd/hub'`を追記。

`setup`のブロックの中にapplicationのホスト`Capybara.app_host = 'http://redmine:3000'`を追記します。

### テスト実行

```sh:Terminal
$ docker-compose -f docker-compose.yml -f docker-compose-dev.yml -f docker-compose.test run --rm redmine bundle exec rake test:system
```

以下のエラーが出て動きませんでした。

```sh:Terminal
NameError: uninitialized constant ApplicationSystemTestCase::Selenium
/redmine/test/application_system_test_case.rb:26:in `<class:ApplicationSystemTestCase>'
/redmine/test/application_system_test_case.rb:22:in `<top (required)>'
```

エラーになった部分を`desired_capabilities: :chrome`に書き換えます。

```ruby:redmine/test/application_system_test_case.rb
  driven_by :selenium, using: :chrome, screen_size: [1024, 900], options: {
      desired_capabilities: :chrome,
      browser: :remote,
      url: 'http://selenium-hub:4444/wd/hub'
    }
```

再度実行。

```sh:Terminal
Run options: --seed 45718

# Running:
...
```

何故か動きました。
エラーの内容は調査しきれな買ったのでまた次回。

テストの実行結果はこんな感じです。

```sh:Terminal
Run options: --seed 45718

# Running:

Capybara starting Puma...
* Version 3.12.1 , codename: Llamas in Pajamas
* Min threads: 0, max threads: 4
* Listening on tcp://127.0.0.1:42831
.[Screenshot]: tmp/screenshots/failures_test_project_quick_search.png
E

Error:
QuickJumpTest#test_project_quick_search:
Capybara::ElementNotFound: Unable to find link or button "Megaproject" within #<Capybara::Node::Element tag="div" path="/HTML/BODY[1]/DIV[1]/DIV[2]/DIV[1]/DIV[2]">
    test/system/quick_jump_test.rb:64:in `block in test_project_quick_search'
    test/system/quick_jump_test.rb:60:in `test_project_quick_search'


bin/rails test test/system/quick_jump_test.rb:54

.E

Error:
MyPageTest#test_add_block:
Net::ReadTimeout: Net::ReadTimeout
    test/system/my_page_test.rb:64:in `test_add_block'

Error:
MyPageTest#test_add_block:
Net::ReadTimeout: Net::ReadTimeout



bin/rails test test/system/my_page_test.rb:57
```

とりあえずテストを動かせるところまでは成功しましたが、テストは失敗しました。
エラーがいっぱい出てます。

でもとりあえず動かせたから今回はここまでにします。

文章にするのは難しいですね。

## 参考

https://qiita.com/yutachaos/items/4a1da5d55a3bf0df889e
