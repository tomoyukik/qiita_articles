# docker-sync.ymlがあるディレクトリをdocker-syncのルートにする

docker-syncを起動するとき、docker-syncルートを指定する方法がわかったので共有します！
いちいちプロジェクトのルートに移動して`docker-sync start`するのが面倒だなって思ってたんですが、やっぱりちゃんと方法はあったんですね。。。

以下のディレクトリ構成で、`volume_dir`をdocker-syncを使ってvolumeします。

```sh:ディレクトリ構成
project_root/
  ┣━ docker-symc.yml
  ┗━ volume_dir/
```

```yml:docker-sync.yml
version: '2'

option:
  project_root: 'config_path'

syncs:
  volume_dir:
    src: './mount_directry'
    sync_strategy: 'unison'
```

こんな感じで`option`要素の`project_root`要素に対して`'config_path'`を指定すれば、`docker-sync.yml`が置いてあるディレクトリをrootとしてdocker-syncが起動できます。
`volume_dir`以下で`docker-sync start`してもちゃんと`volume_dir`がvolumeできるわけです。

この`project_root`要素では`'pwd'`と`'config_path'`のどちらかを指定できるみたいなんですが、デフォルトでは`'pwd'`が指定されています。なので、カレントディレクトリがルートになってしまいます。

どういう時に`'pwd'`を使うのか思いつかないんですが、もし知ってる方がいたら教えて欲しいです。

## 参考

docker-syncはgemだし`src: '<%= Dir.pwd %>/mount_directory'`とかでできるんじゃないかと思ったけど全然違いました。。。

[`docker-sync`のgithubリポジトリ](https://github.com/EugenMayer/docker-sync)にサンプルがあったんですね。。。もっと早く知りたかった。

他にも色々細かく設定できるみたいです。時間があるときに日本語訳します。（[参考](https://github.com/EugenMayer/docker-sync/blob/master/example/docker-sync.yml)）

```yml:https://github.com/EugenMayer/docker-sync/blob/master/example/docker-sync.yml
# version 0.2.x以降から記述が必須
version: "2"

# もっと簡単な例は https://github.com/EugenMayer/docker-sync-boilerplate の雛形を参照
options:
  # 変更可能。デフォルトでは docker-compose.yml 。指定するcomposeファイルの場所とかパスは変更可能
  compose-file-path: 'docker-compose.yml'
  # 変更可能。デフォルトは docker-compose-dev.yml 。指定するcomposeファイルの場所とかパスは変更可能。使用しない場合はファイルを配置しないこと
  # 指定したcomposeファイルが存在する場合は使われる。ファイルの名前を明示的に指定する場合はファイルが存在してないとだめ
  compose-dev-file-path: 'docker-compose-dev.yml'
  # 選択可能。デバッグが必要な場合はtrueに設定する。デフォルトはfalse
  verbose: true
  # can be docker-sync, thor or auto and defines, how the sync will be invoked on the cli. Mostly depending if your are using docker-sync solo, scaffolded or in development ( thor )
  cli_mode: 'auto'
  # optional, maximum number of attempts for unison waiting for the success exit status. The default is 5 attempts (1-second sleep for each attempt). Only used in unison.
  max_attempt: 5
  # optional, default: pwd, root directory to be used when transforming sync src into absolute path, accepted values: pwd (current working directory), config_path (the directory where docker-sync.yml is found)
  project_root: 'pwd'

  # replace <sync_strategy> with either rsync, unison, native_osx to set a custom image for all sync of this type
  <sync_strategy>_image: 'yourcustomimage'
syncs:
  default-sync:
    # if you want to run a custom image, set it here
    image: 'yourcustomimage'
    # os aware sync strategy, default to unison under osx, and native docker volume under linux
    sync_strategy: 'default'
    # which folder to watch / sync from - you can use tilde, it will get expanded.
    # the contents of this directory will be synchronized to the Docker volume with the name of this sync entry ('default-sync' here)
    src: './default-data/'

    # this is only available if you use docker-for-mac edge for now
    host_disk_mount_mode: 'cached' # see https://github.com/moby/moby/pull/31047
    # other unison options can also be specified here, which will be used when run under osx,
    # and ignored when run under linux

  unison-sync:
    # unison 2 way-sync
    sync_strategy: 'unison'
    # common options
    # see rsync documentation for all these common options
    src: './data4/'
    # be aware, this only gives you a notification on the initial sync, not the syncs after changes. this is a difference
    # to the rsync implementation
    notify_terminal: true
    # default is 'auto', which means, your docker-machine/docker host ip will be detected automatically. If you set this to a concrete IP, this ip will be enforced
    sync_host_ip: 'auto'    # when a port of a container is exposed, on which IP does it get exposed. Localhost for docker for mac, something else for docker-machine
    sync_userid: '33'
    # specific options
    # If you need to sync a lot of files, you can reach out the system limit
    # of inotify watches. You can set the limit by using this parameter. This will
    # prompt you for your sudo password to modify the system configuration.
    #max_inotify_watches: 100000

    # Additional unison options
    # @see http://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html#prefs
    # For example, these provided options will automatically resolve conflicts by using the newer version of the file
    # and append a suffix to the conflicted file to prevent its deletion.
    # do not use --copyonconflict or --prefer here, those are handled by the sync_prefer setting
    sync_args: [ '-v' ]
    # Exclude some files / directories that matches **exactly** the path
    # this currently use the the -Path option of unison, use sync_excludes_type to change this behavior
    # see http://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html#pathspec for more
    sync_excludes: [ '.git', '.idea', 'node_modules' ]

    # use this to change the exclude syntax.
    # Path: you match the exact path ( nesting problem )
    # Name: If a file or a folder does match this string ( solves nesting problem )
    # Regex: Define a regular expression
    # none: You can define a type for each sync exclude, so sync_excludes: ['Name .git', 'Path Gemlock']
    #
    # for more see http://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html#pathspec
    # Name is the default since 0.2.0
    sync_excludes_type: 'Name'

    # defines how sync-conflicts should be handled. With default it will prefer the source with --copyonconflict
    # so on conflict, pick the one from the host and copy the conflicted file for backup
    sync_prefer: 'default'

  rsync-sync: # IMPORTANT: this name must be unique and should NOT match your real application container name!
    # enable terminal_notifier. On every sync sends a Terminal Notification regarding files being synced. ( Mac Only ).
    # good thing incase you are developing and want to know exactly when your changes took effect.
    notify_terminal: true
    # which folder to watch / sync from - you can use tilde, it will get expanded.
    # the contents of this directory will be synchronized to the Docker volume with the name of this sync entry ('rsync-sync' here)
    src: './data1/'
    # when a port of a container is exposed, on which IP does it get exposed. Localhost for docker for mac, something else for docker-machine
    # default is 'auto', which means, your docker-machine/docker host ip will be detected automatically. If you set this to a concrete IP, this ip will be enforced
    sync_host_ip: 'auto'
    # should be a unique port this sync instance uses on the host to offer the rsync service on
    sync_host_port: 20871
    # set of IPs (10.0.2.2) or networks (10.0.0.0/8) that will be granted access to this rsync service, useful when running on FreeBSD with Virtualbox and a host-only interface
    # sync_host_allow: '10.0.0.0/8'
    # optionl, a list of excludes for rsync - see rsync docs for details
    sync_excludes: ['Gemfile.lock', 'Gemfile', 'config.rb', '.sass-cache/', 'sass/', 'sass-cache/', 'composer.json' , 'bower.json', 'package.json', 'Gruntfile*', 'bower_components/', 'node_modules/', '.gitignore', '.git/', '*.coffee', '*.scss', '*.sass']
    # optional: use this to switch to rsync verbose mode
    sync_args: '-v'
    # optional, a list of regular expressions to exclude from the fswatch - see fswatch docs for details

    # use rsync by setting this
    sync_strategy: 'rsync'

    # this does not user groupmap but rather configures the server to map
    # optional: usually if you map users you want to set the user id of your application container here
    # set it to 'from_host' to automatically bound it to the uid of the user who launches docker-sync (unison only at the moment)
    sync_userid: '5000'
    # optional: usually if you map groups you want to set the group id of your application container here
    # this does not user groupmap but rather configures the server to map
    # this is only available for unison/rsync, not for d4m/native (default) strategies
    sync_groupid: '6000'

    # optional: enable fswatch in the container side (unison only) to automatically retrieve files created in the container to the host
    watch_in_container: true
    watch_excludes: ['.*/.git', '.*/node_modules', '.*/bower_components', '.*/sass-cache', '.*/.sass-cache', '.*/.sass-cache', '.coffee', '.scss', '.sass', '.gitignore']
    # optional: use this to switch to fswatch verbose mode
    watch_args: '-v'
    # optional: default is fswatch, if set to disable, no watcher will be used and you would need to start the sync manually
    watch_strategy: 'fswatch'

    # monit can be used to monitor the health of unison in the native_osx strategy and can restart unison if it detects a problem
    # optional: use this to switch monit monitoring on
    monit_enable: false
    # optional: use this to change how many seconds between each monit check (cycle)
    monit_interval: 5
    # optional: use this to change how many consecutive times high cpu usage must be observed before unison is restarted
    monit_high_cpu_cycles: 2
```

