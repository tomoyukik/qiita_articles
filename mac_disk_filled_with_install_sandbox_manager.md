# Macのストレージ「その他」が圧迫問題

Macのストレージを圧迫していた`.PKInstallSandboxManager`について調べていたものの、中々情報が見つけられなかったので、情報共有の意味合いを兼ねてかいた記事です。
`.PKInstallSandboxManager`が何かまでは書いてません。

何を書いたかというと、`/Library/InstallerSandboxes/.PKInstallSandboxManager/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.activeSandbox`というディレクトリがストレージを圧迫していたんですが、このディレクトリはXcodeのアップデートの時に作成されるみたいで、アップデートが終わると自動で削除されるみたいだよということです。

何に使われているのか、削除していいものなのかまでは分かりませんが、僕は迷った末に消しました。
問題は今のところ起きていませんが、削除する場合は自己責任で実行してください。

## 詳細

何があってどんなことしたかを少し詳細に書いているだけです。

### ストレージの圧迫

私用のMacでディスクが容量がいっぱいになっている旨のワーニングが出たので、ストレージの内訳を確認すると150G / 250Gが「その他」の項目で占められていました。
最近ストレージを空けたばっかりだったのと、全く思い当たる節がなかったので、[DaisyDisk](https://daisydiskapp.com)を購入して詳細を調べると、`/Library/InstallerSandboxes/.PKInstallSandboxManager`というディレクトリ以下に15G程の`XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.activeSandbox`というディレクトリを6個ほど発見。合計で「その他」の90G程を占めていました。
どのディレクトリも同様の構成になっていて、なんとなくXcodeで使われていそうなことまでは分かったものの、消していいものか判断がつかず。。。
迷った末に[消している人がいた](https://ohwhsmm7.blog.fc2.com/blog-entry-586.html)ので、結局消すことにしました。

因みにですが、削除するときはコンソールの操作間違いが怖かったので、`sudo open /Library/InstallerSandboxes/.PKInstallSandboxManager`でファインダーを開き、GUI操作でゴミ箱に移動しました。


### `activeSandbox`ディレクトリの挙動

XcodeをAppStoreからアップデートしている際に`/Library/InstallerSandboxes/.PKInstallSandboxManager`以下を確認すると、同様の`XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.activeSandbox`というディレクトリが作成されていたのですが、アップデートが終わると削除されていました。
アップデートにも削除にも時間がかかるので、アップデートの実行中にストレージ内訳を見ていると、少しずつ空き容量が減っていって、途中から空き容量が増えていく様子が分かります。

つまり、消していいものかは分からないけど、普通は残らず消えそうだねということは分かりました。

## DaisyDiskについて

調べれば出てくると思うので詳細は書きませんが、ディスク使用量の内訳を可視化してくれる有料のアプリです。
「このMacについて」からでは分からない「その他」項目の詳細も分かります。

AppStoreからもインストールできますが、AppStoreからインストールすると管理者権限での実行ができず、`/Library/InstallerSandboxes`のような隠し領域の可視化ができません。
もしAppStoreからインストールしてしまった場合は、DaisyDiskのページからFREE TRIAL版をインストールすると、AppStoreのライセンスを自動で認識してくれるようです ( https://softantenna.com/wp/mac/daisy-disk-3-0/ )。

僕はAppStoreでインストールしてしまったので、FREE TRIAL版をインストールしなおしました。

## 参考

- https://ohwhsmm7.blog.fc2.com/blog-entry-586.html (ストレージの空き容量を増やす)
- https://daisydiskapp.com (DaisyDisk)
- https://softantenna.com/wp/mac/daisy-disk-3-0/ (DaisyDiskについて)

