# DockerでRedmineのSystem Testを動かしてみた

## Dockerで環境を整える

```
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

```
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

```
#!/bin/bash
set -e

rm -f /redmine/tmp/pids/server.pid

exec "$@"
```

## chrome側からRedmineをみてみる

## RedmineのTestファイルを修正

```
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



なんか動かなかった
```
NameError: uninitialized constant ApplicationSystemTestCase::Selenium
/redmine/test/application_system_test_case.rb:26:in `<class:ApplicationSystemTestCase>'
/redmine/test/application_system_test_case.rb:22:in `<top (required)>'
```

```
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



## テストを実行してみる

```
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

なんかいっぱいエラー出たけどとりあえず動かせたから今回はここまで。
どうしてエラーが出てるのかわかる方いたら教えてください


## 参考

https://qiita.com/yutachaos/items/4a1da5d55a3bf0df889e
docker公式のページ
selenium-hubのページ
