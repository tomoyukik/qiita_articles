# `homebrew`でインストールした`Ruby`と`gem`にパスを通す

今まで`Ruby`のインストールには`rbenv`を使ってたんですが、
最近は`Docker`を使ってるので、MacBookの新調をきっかけに`rbenv`をやめて、`homebrew`でシステムインストールしました。
`homebrew`でインストールした`Ruby`を使うには、
`Ruby`と`gem`それぞれに対してパスを通してあげないといけないみたいです。

僕は`fish`を使ってるのでこんな感じです。

```fish:~/.config/fish/config.fish
set PATH /usr/local/lib/ruby/gems/2.6.0/bin /usr/local/opt/ruby/bin 
```

`bash`?の人はこんな感じでしょうか。

```sh:~/.bashrc
export PATH=$PATH:/usr/local/lib/ruby/gems/2.6.0/bin:/usr/local/opt/ruby/bin 
```

- `/usr/local/opt/ruby/bin`
  - `Ruby`のパス
- `/usr/local/lib/ruby/gems/2.6.0/bin`
  - `gem`のパス

です。

## 蛇足

`homebrew`でシステムインストールすると言っても、`homebrew`をインストールするとMacデフォルトの`Ruby`とは別に`Ruby`がインストールされるみたいです。

```sh:Terminal
><((("> brew install ruby
Warning: ruby 2.6.3 is already installed and up-to-date
```

まずハマったのがここなんですが、`Ruby`のバージョンを確認すると`homebrew`でインストールしたものとは異なりました。

```sh:Terminal
><((("> ruby --version
ruby 2.3.7p456 (2018-03-28 revision 63024) [universal.x86_64-darwin18]
><((("> which ruby
/usr/bin/ruby
```

このままではデフォルトの`Ruby` を参照しているのでパスを通してあげないといけないみたいです。

```fish:~/.config/fish/config.fish
set PATH /usr/local/opt/ruby/bin
```

```sh:Terminal
><((("> source ~/.config/fish/config.fish
><((("> ruby --version
ruby 2.6.3p62 (2019-04-16 revision 67580) [x86_64-darwin18]
```

一件落着かと思いきや、確かにインストールしたはずの`docker-sync`に対してもパスが通ってません。

```sh:Terminal
><((("> gem install docker-sync
Successfully installed docker-sync-0.5.11
Parsing documentation for docker-sync-0.5.11
Done installing documentation for docker-sync after 0 seconds
1 gem installed
><((("> docker-sync start
fish: Unknown command docker-sync
```

`gem`は`gem`でパスを通してあげないといけないみたいですね。

```fish:~/.config/fish/config.fish
set PATH /usr/local/lib/ruby/gems/2.6.0/bin /usr/local/opt/ruby/bin
```

これで本当に解決です。

## 余談

なんとなく、ほんの1年前まで「パスを通す」の意味もわからなかったことを思い出しました。
こんな記事でも誰かの参考になればいいなと思います。
もしわからないことあったらコメントとかで聞いてください。
間違いとかも指摘してもらえたらと思います。

それにしてもカスタマイズに時間が取られすぎてやりたいことに手がつけられません。。。

## 参考

[StackOverflow](https://stackoverflow.com/questions/6482738/installing-ruby-gems-not-working-with-home-brew)
