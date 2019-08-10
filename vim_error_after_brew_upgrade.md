# `brew upgrade vim`したら`vim`が開けなくなったときの対処

ふと思い立ってvimをhomebrewでupgrade。

```sh:Terminal
$ brew upgrade vim
```

そしてvimを開こうとしたらエラー。

```sh:Terminal
$ vim
dyld: Library not loaded: /usr/local/opt/ruby/lib/libruby.2.6.dylib
  Referenced from: /usr/local/bin/vim
  Reason: image not found
fish: '/usr/local/bin/vim --version' terminated by signal SIGABRT (Abort)
```

vimが開けない！！
`/usr/local/opt/ruby/lib/libruby.2.6.dylib`がロードできないっていってるけど
そもそもそこにディレクトリなんかない。

諦めようと思ったけど、最近vimのカスタマイズを始めたところだったのでなんとかしたい！
ということで調べてみた。

macで結構よく起きてる問題みたいで、`brew reinstall`をするといいみたいなので実行。

```sh:Terminal
$ brew reinstall vim
==> Reinstalling vim
==> Installing dependencies for vim: ruby
==> Installing vim dependency: ruby
==> Downloading https://homebrew.bintray.com/bottles/ruby-2.6.3.mojave.bottle.tar.gz
Alr-only, which means it was not symlinked into /usr/local,
 48 because macOS already provides this software and installing another version in
 49 parallel can cause all kinds of trouble.eady downloaded: /Users/tomoyukik/Library/Caches/Homebrew/downloads/cfb64f0fc9206273e73b1436cee1b0577518edc20123f54db09f64883bcea1de--ruby-2.6.3.mojave.bottle.tar.gz
==> Pouring ruby-2.6.3.mojave.bottle.tar.gz
==> Caveats
By default, binaries installed by gem will be placed into:
  /usr/local/lib/ruby/gems/2.6.0/bin

You may want to add this to your PATH.

ruby is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.

If you need to have ruby first in your PATH run:
  echo 'set -g fish_user_paths "/usr/local/opt/ruby/bin" $fish_user_paths' >> ~/.config/fish/config.fish

For compilers to find ruby you may need to set:
  set -gx LDFLAGS "-L/usr/local/opt/ruby/lib"
  set -gx CPPFLAGS "-I/usr/local/opt/ruby/include"

For pkg-config to find ruby you may need to set:
  set -gx PKG_CONFIG_PATH "/usr/local/opt/ruby/lib/pkgconfig"

==> Summary
🍺  /usr/local/Cellar/ruby/2.6.3: 19,372 files, 32.4MB
==> Installing vim
==> Downloading https://homebrew.bintray.com/bottles/vim-8.1.1800.mojave.bottle.tar.gz
Already downloaded: /Users/tomoyukik/Library/Caches/Homebrew/downloads/c8d28ec31c1c2451358735556fb5a0c52d863661f99646dde0cbf94893471603--vim-8.1.1800.mojave.bottle.tar.gz
==> Pouring vim-8.1.1800.mojave.bottle.tar.gz
🍺  /usr/local/Cellar/vim/8.1.1800: 1,859 files, 31.6MB
==> Caveats
==> ruby
By default, binaries installed by gem will be placed into:
  /usr/local/lib/ruby/gems/2.6.0/bin

You may want to add this to your PATH.

ruby is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.

If you need to have ruby first in your PATH run:
  echo 'set -g fish_user_paths "/usr/local/opt/ruby/bin" $fish_user_paths' >> ~/.config/fish/config.fish

For compilers to find ruby you may need to set:
  set -gx LDFLAGS "-L/usr/local/opt/ruby/lib"
  set -gx CPPFLAGS "-I/usr/local/opt/ruby/include"

For pkg-config to find ruby you may need to set:
  set -gx PKG_CONFIG_PATH "/usr/local/opt/ruby/lib/pkgconfig"
```

`reinstall`しても結局vimは開けない。
でも`reinstall`を実行したときのメッセージをよく読むと

```sh:Terminal
ruby is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.
```

って書いてある。
「*keg-only*だから`/usr/local`シンボリックリンクは自動で貼られないよ」って意味だと思う。
（しばらく意味がわからなくて途方にくれた。）

だから`ruby`のシンボリックリンクを`/usr/local`以下に貼ればいいんじゃないかな？

僕は`rbenv`を使ってるのでrubyのパスはここ。

```sh:Terminal
$ which ruby
/Users/tomoyukik/.rbenv/shims/ruby
```

だからこの`ruby`に対してシンボリックリンクを貼ってみた。

```sh:Terminal
$ ln -s /Users/tomoyukik/.rbenv/shims/ruby  /usr/local/bin/ruby
```

vim開けた！


...けど警告がなぜかいっぱい出る。

```sh:Terminal
$ vim
Warning: Failed to set locale category LC_NUMERIC to en_JP.
Warning: Failed to set locale category LC_TIME to en_JP.
Warning: Failed to set locale category LC_COLLATE to en_JP.
Warning: Failed to set locale category LC_MONETARY to en_JP.
Warning: Failed to set locale category LC_MESSAGES to en_JP.
```

調べたらこれもよく起きてるみたいで、localeの設定を`.bash_profile`に書けばいいって書いてある。
僕は`fish`を使ってるので`~/.config/fish/config.fish`にlcoaleの設定を付け足してみた。

```go:~/.config/fish/config.fish
set -x set LC_ALL en_JP
```

```sh:Terminal
$ source ~/.config/fish/config.fish
```

そしたら警告が増えた！

```sh:Terminal
$ vim
Warning: Failed to set locale category LC_NUMERIC to en_JP.
Warning: Failed to set locale category LC_TIME to en_JP.
Warning: Failed to set locale category LC_COLLATE to en_JP.
Warning: Failed to set locale category LC_MONETARY to en_JP.
Warning: Failed to set locale category LC_MESSAGES to en_JP.
Warning: Failed to set locale category LC_NUMERIC to en_JP.
Warning: Failed to set locale category LC_TIME to en_JP.
Warning: Failed to set locale category LC_COLLATE to en_JP.
Warning: Failed to set locale category LC_MONETARY to en_JP.
Warning: Failed to set locale category LC_MESSAGES to en_JP.
```

もうわけがわからない...
そもそもカッコつけて英語でmac使ってたなあと思って`locale`を確認。

```sh:Terminal
$ locale
LANG=
LC_COLLATE="C"
LC_CTYPE="C"
LC_MESSAGES="C"
LC_MONETARY="C"
LC_NUMERIC="C"
LC_TIME="C"
LC_ALL="C"
```

`"C"`がなんだかよくわからない...
とりあえずvimは使えるので今回は一旦諦める。。。

`/usr/local/opt/ruby`と`/usr/local/bin/ruby`で参照してるrubyが違うのが気になってるけど関係あるのかな...？

```sh:Terminal
$ ls -la /usr/local/opt/ | grep ruby
lrwxr-xr-x   1 tomoyukik  admin    20 Aug  8 22:57 ruby -> ../Cellar/ruby/2.6.3
lrwxr-xr-x   1 tomoyukik  admin    29 Jul 28 09:31 ruby-build -> ../Cellar/ruby-build/20190615
lrwxr-xr-x   1 tomoyukik  admin    20 Aug  8 22:57 ruby@2.6 -> ../Cellar/ruby/2.6.3
$ ls -la /usr/local/bin/ruby
lrwxr-xr-x  1 tomoyukik  admin  37 Aug  8 23:03 /usr/local/bin/ruby -> /Users/tomoyukik/.rbenv/shims/ruby
```

誰かわかる人いたら助けてください！

参考までにmacのバージョン。

```sh:Terminal
$ sw_vers
ProductName:	Mac OS X
ProductVersion:	10.14.5
BuildVersion:	18F132
```
