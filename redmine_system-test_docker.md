# DockerでRedmineのSystem Testを動かしてみた



## Dockerで環境を整える

まず、ディレクトリの構成は以下。
`redmine`はここ[https://github.com/redmine/redmine]からclone。
`postgres`ディレクトリは、`docker-compose`でコンテナを起動すれば勝手に作成されるはず。
`postgres`のデータを保存する場所です。
`postgres`の中は空じゃないと`postgresql`のコンテナの起動に失敗するので注意です。

```ディレクトリの構成:sh
workspace/
 ├── docker-compose.yml
 ├── Dockerfile
 ├── entrypoint.sh
 ├── redmine/
 └── postgers/
```

各ファイルの内容です。

`redmine`は`Dockerfile`からビルド。
`selenium-hub`は何だろう。。。
`chrome`がchromeドライバです。
redmineのシステムテストを動かすために使う。

```docker-compose.yml:yml
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
      - chrome
  postgres:
    image: postgres:9.6
    volumes:
      - ./postgres:/var/lib/postgresql/data
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

```entrypoint.sh:sh
#!/bin/bash
set -e

rm -f /redmine/tmp/pids/server.pid

exec "$@"
```

## chrome側からRedmineが見えるか確認

ブラウザを立ち上げてアドレスバーに、`http://localhost:5900`を入力。
画面共有をするか聞かれるので`OK`を選択。
パスワードを聞かれたら`secret`。
黒い画面が立ち上がる。
右クリック -> Application -> Network -> chrome。
立ち上げたchromeのアドレスバーに`http://redmine:3000`を入力。
`redmine`は`docker-compose.yml`で指定したサービス名。
Redmineの画面が表示されたので成功。
先に進む。

## RedmineのTestファイルを修正

Redmineのシステムテストの設定を変更。
「「「「ここ」」」」を参考にしつつ、SystemTestingのソースを見てブラウザの設定とurlを足してみる。
chromeから接続する先をsetupの中に記述。


```redmine/test/application_system_test_case.rb:ruby
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

## テストを実行してみる

`docker-compose exec redmine bundle exec rake test:system`で実行。

動かなかった。

```terminal:sh
NameError: uninitialized constant ApplicationSystemTestCase::Selenium
/redmine/test/application_system_test_case.rb:26:in `<class:ApplicationSystemTestCase>'
/redmine/test/application_system_test_case.rb:22:in `<top (required)>'
```

エラーの内容はよくわからなかったけど、とりあえずエラーになった部分をコメントアウト。
「「「ここ」」」を参考に書き換えてみる。
とりあえず、`desired_capabilities`を書き換えるとうまくいった。
調査は後回し。

```redmine/test/application_system_test_case.rb:ruby
  driven_by :selenium, using: :chrome, screen_size: [1024, 900], options: {
      desired_capabilities: :chrome,
#       desired_capabilities: Selenium::WebDriver::Remote::Capabilities.chrome(
#         'chromeOptions' => {
#           'prefs' => {
#             'download.default_directory' => DOWNLOADS_PATH,
#             'download.prompt_for_download' => false,
#             'plugins.plugins_disabled' => ["Chrome PDF Viewer"]
#           }
#         }
#       ),
      browser: :remote,
      url: 'http://selenium-hub:4444/wd/hub'
    }
```

再度実行。

```terminal:sh
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

動いたけど、テストは失敗。
なんかいっぱいエラー出たけどとりあえず動かせたから今回はここまで。
どうしてエラーが出てるのかわかる方いたら教えてください


## 参考

https://qiita.com/yutachaos/items/4a1da5d55a3bf0df889e
docker公式のページ
selenium-hubのページ
