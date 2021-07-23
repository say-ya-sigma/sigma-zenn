---
title: "スマホでコード書く 環境構築編2"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux"]
published: false
---

スマホでコードを書く意義、それは主にテクノロジーへのアクセシビリティ向上にあります。スマホは既に日本人にとって最も馴染みのあるコンピュータデバイスです。その上で様々なテクノロジーに無料で触れるできればどれほどの価値があるでしょうか。

## golangを使う

「移植性の高い最近の言語といえばgo。」という安直な理由で、スマホでコードを書く際にもgoを採用します。良くも悪くも基本的な道具しかないので、実験的な学習用プロジェクトにもピッタリな言語だと思います。

Ubuntuのgoはパッケージが古いため、公式から最新バージョンをダウンロードしてインストールします。

https://golang.org/dl/

```bash
wget https://golang.org/dl/go1.16.6.linux-arm64.tar.gz
```

インストールは解凍して配置するだけです。

```bash
tar -C /usr/local -xzf go1.16.6.linux-arm64.tar.gz
```

`.bashrc`でパスを追記します。

```bash
export PATH=$PATH:/usr/local/go/bin
```

## node.jsどうするか問題

これまでのスマホローカルでの開発では、node.jsどうするか問題が常につきまとっていましたが、この問題はnode.jsのARM対応が急速に改善されるという形で現在ではほぼ解決しています。16系ではAppleのM1向けのバイナリが提供され、エンジニアからのより多くのフィードバックを得て改善が進むことが予想されます。

今回は14系の最新版をインストールしようと思います。Ubuntuのnodejsパッケージとnpmパッケージをインストールした後、node.jsのバージョンマネージャーnをインストールして14系の最新版をインストールします。

```bash
sudo apt install nodejs npm
npm i -g n
n i 14.17.3
```

## MongoDBをインストール

データベースは公式ページでARM対応を謳っているMongoDBをインストールします。スキーマレス最高！巷では物議を醸しがちなMongoDBですが自分は結構好きです。近年ではマネージドサービスも充実してきています。

MongoDBは初心者向けというわけではないです。少なくとも業務で投入する上では様々な落とし穴に注意を払う必要があると思います。オススメをするわけではないということに注意してください。

https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/

ARM版Ubuntu向けのリポジトリがあるのでそこからインストールします。バージョンは5.0.0です。

```bash
echo "deb [ arch=arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

コンフィグを書きます。開発環境ではDBの中身を全て消すこともあり得るので扱いやすい位置にdbディレクトリを置きそこを指定します。

```yaml
storage:
   dbPath: /home/sigma/develop/hello_task/mongodb/db
systemLog:
   destination: file
   path: /home/sigma/develop/hello_task/mongodb/log/mongod.log
   logAppend: true
```

ログを漁ることは少ないと思いますが、バックグラウンドで動作させるのに必要なのでファイルを指定します。

## ディレクトリ構成

ディレクトリ構成を考えます。まず作るサービスの名前を決めます。todoアプリを拡張するようなイメージで作って行くので`hello_task`という名前をつけます。

```bash
hello_task$ tree -L 2
```

```
.
├── backend
│   ├── controller
│   ├── go.mod
│   ├── go.sum
│   ├── main.go
│   ├── repository
│   └── service
├── devscript
│   ├── close.sh
│   └── start.sh
├── frontend
│   ├── README.md
│   ├── assets
│   ├── components
│   ├── jest.config.js
│   ├── jsconfig.json
│   ├── layouts
│   ├── node_modules
│   ├── nuxt.config.js
│   ├── package-lock.json
│   ├── package.json
│   ├── pages
│   ├── static
│   ├── store
│   ├── test
│   └── tsconfig.json
└── mongodb
    ├── db
    ├── log
    └── mongod.yaml

17 directories, 13 files
```

frontendとbackendに分けます。それぞれ別のgitリポジトリとして管理します。frontendはnuxtを使うので、ディレクトリ構造もnuxtに従います。backendはシンプルにcontroller service repositoryに切り分けます。ドメインロジックは可能な限りserviceに凝集します。

MongoDBの実体やログデータなど格納を格納するmongodbディレクトリを準備します。

## スクリプト作成

最後に起動スクリプトを書きます。devscriptというディレクトリを切ります。

```bash
DEVDIR=~/develop/hello_task
source $DEVDIR/devscript/close.sh
tmux new-window -d -n nuxt -c $DEVDIR/frontend "npm run dev"
mongod --fork --config $DEVDIR/mongodb/mongod.yaml >> /dev/null
tmux new-window -d -n gin -c $DEVDIR/backend "go run main.go"
```

```bash
tmux kill-window -t nuxt
tmux kill-window -t gin
mongo --eval "db.getSiblingDB('admin').shutdownServer()" >> /dev/null
```

