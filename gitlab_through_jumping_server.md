# 踏み台サーバを経由してGitLabをホストした

踏み台サーバを経由した時のGitLabの設定が想定よりハマりました。その時の設定方法のメモ書きです。  
`ssl`とか`ssh`については今回は設定していません。

## 構成

今回の構成です。

![[Laptop PC] -> 8888[踏み台] -> 8080[GitLab Server 80[GitLab Docker]]](./gitlab_through_jumping_server/servers.drawio.svg)

- GitLabはリモートサーバ上でDockerを使って起動します。
- リモートサーバは`8080`ポートをDockerコンテナの`80`番にフォワードします。
- 踏み台サーバは`8888`番で受けたリクエストをリモートサーバの`8080`番にSSHでトンネリングします。

## 手順

### 1. リモートサーバでのGitLabの起動

今回はGitLabのDocker Imageを使用しました。

まずは`docker-compose.yaml`の作成です。

```yaml:docker-compose.yaml
version: '3.9'

services:
    image: gitlab/gitlab-ee:15.1.2-ee.0
    ports:
      - 8080:80
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://ip.address.of.jump:8888'
        nginx['listen_port'] = 80
    volumes:
      - ./config:/etc/gitlab
      - ./logs:/var/log/gitlab
      - ./data:/var/opt/gitlab
    restart: always
```

- リモートサーバの`8080`ポートをDockerコンテナの`80`番に割り当てます。
- `external_url`はGitLabのUIに表示されるURLです。 　
  踏み台サーバを経由してアクセスするため、踏み台サーバのIPアドレス`ip.address.of.jump`とポート`8888`を使用します。
- `nginx['listen_port']`は`80`番を使用します。　 
  この場合、明示的に指定しないと`external_url`の`8888`番が使われてしまいます (間違っていたらすみません)。

下記コマンドで起動します。

```sh user@remote$ docker compose up -d ```

GitLabの準備は完了です。

### 2. 踏み台サーバでのポートフォワード

続いて`ssh`コマンドを用いてポートフォワードを行います。  

とりあえず踏み台サーバで鍵ペアを作成してリモートサーバに配置しましょう。  
詳しくは省略しますが、踏み台サーバ上で`ssh-keygen`で鍵を作成し、公開鍵`some_key.pub`の値をリモートサーバの`.ssh/authorized_key`に書き込みましょう。

下記コマンドでポートフォワードを行いました。

```sh
nohup ssh -N -p 22 someuser@ip.address.of.remote -L 8888:localhost:8080 -g &
```

- `nohup ... &`バックグラウンドでコマンドを実行するためのコマンドです。
- `-N`はリモートコマンドを実行しないためのオプションです。  
  `ssh -N -p 22 someuser@ip.address.of.remote -L 8888:localhost:8080 -g`を実行するとオプションの意味がわかるでしょう。
- `-L`はローカルフォワードのオプションです。　 
  ローカル(踏み台)の`8888`ポートをリモート(`ip.address.of.remote`) の`8080`ポートにフォワードしています。  
  この時、`-L 8888:localhost:8080`の`localhost`はssh先、つまりリモートサーバ上での`localhost`を指すようです ([参考](https://qiita.com/mechamogera/items/b1bb9130273deb9426f5)) 。  
- `-g`オプションをつけないと、踏み台サーバからのアクセスしかフォワードされません。

### 3. ブラウザからアクセス

ブラウザに `http://ip.address.of.jump:8888`を打ち込んでアクセスしてください。  
`502`が表示されるときはもう少し待ちましょう。

テキトーなリポジトリのクローン先のURLを確認しましょう。
踏み台サーバのアドレスになっていればOKです。

TANUKIアイコンが少したぬきらしくなりましたね。

## おわり

以上、踏み台サーバを経由してGitLabをホストする方法でした。  

思い込みや勘違いのお陰で大層はまってしまいました。  
ちゃんとドキュメントなどで正確な情報を確認するようにしましょうね。

不正確な情報が含まれている可能性があるので、

- `nginx[listen_port]`を指定しないときのポートの値
- `nohup ssh -N -p 22 someuser@ip.address.of.remote -L 8888:localhost:8080 -g &`の`localhost`

辺りは本当かどうか確認する必要がありそうです。