---
title: "【Rails7】DockerでRuby on Railsアプリを構築してHerokuにデプロイするまで"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails","Docker","Heroku"]
published: true
---

こんにちは。Hisayaです。
今回は、DockerでRuby on Railsアプリを構築してHerokuにデプロイするまでの手順を解説を交えながら（備忘録も兼ねて）記事に残したいと思います。

# 環境

今回使用する環境は下記の通りです。

- Docker
- Ruby: 3.0.2
- Rails 7.0.1
- Node.js: 15.x

# 必要なファイルの用意

## 1. Railsアプリの作成

まずはRailsアプリを作るために必要なファイルを作成します。
この部分は本題と逸れるのでサクッと説明します。

```bash
$ mkdir my-app
$ cd my-app
$ bundle init
Writing new Gemfile to /path/to/my-app/Gemfile
```

`Gemfile`ができるので、Railsをインストールするように編集します。

```Gemfile:Gemfile
# frozen_string_literal: true
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

gem "rails", "~> 7.0.0"
```

次に`bundle install`を実行して`Gemfile.lock`を生成します。

```bash
$ bundle install --path vendor/bundle
```

`rails new`をしてRailsアプリの雛形を作ります。
※ databaseに`postgresql`を指定していますが、MySQLを使用する場合は`mysql`を指定してください。

```
$ echo y | bundle exec rails new . -d postgresql --skip-bundle --skip-turbolinks --skip-system-test
```

このままだとHerokuにデプロイした際にBlocked hostエラーが発生するので、`config/environments/development.rb`に下記を追記します。

```ruby:config/environments/development.rb
Rails.application.configure do
  ...
  config.hosts.clear
end
```

## 2. `Dockerfile`

```Dockerfile:Dockerfile
FROM ruby:3.0.2

RUN set -x \
  && curl -sL https://deb.nodesource.com/setup_15.x | bash - \
  && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
  && apt-get update -qq \
  && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev libxslt-dev libxml2-dev \
    nodejs yarn \
    curl vim sudo cron \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* \
  && rm -rf /var/cache/yum/*

# set environment variables
ENV LANG C.UTF-8
ENV APP_ROOT /app
ENV BUNDLE_JOBS 4
ENV BUNDLER_VERSION 2.2.25

# working on app root
RUN mkdir /app
WORKDIR $APP_ROOT
COPY Gemfile $APP_ROOT/Gemfile
COPY Gemfile.lock $APP_ROOT/Gemfile.lock
RUN gem install bundler -v $BUNDLER_VERSION
RUN bundle -v
RUN bundle install
COPY . $APP_ROOT

# execute script after build container
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000
```

### 補足

たくさんインストールしてますが、この辺はお好みで。

```Dockerfile
RUN set -x \
  && curl -sL https://deb.nodesource.com/setup_15.x | bash - \
  && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
  && apt-get update -qq \
  && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev libxslt-dev libxml2-dev \
    nodejs yarn \
    curl vim sudo cron \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* \
  && rm -rf /var/cache/yum/*
```

今回は`entrypoint.sh`でゴニョゴニョやりたいので、`ENTRYPOINT`を設定しています。

```Dockerfile
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
```

## 3. `entrypoint.sh`

```bash:entrypoint.sh
#!/bin/bash

set -ex
rm -f /app/tmp/pids/server.pid

if [ $RAILS_ENV = 'production' ]; then
  bundle exec rails assets:clobber
  bundle exec rails assets:precompile
fi

bundle exec rails server -b '0.0.0.0' -p ${PORT:-3000}
```

[Heroku公式](https://devcenter.heroku.com/ja/articles/container-registry-and-runtime#unsupported-dockerfile-commands)にも書いてありますが、HerokuにDockerイメージをデプロイする場合は環境変数（`PORT`）からポート番号を取得する必要があります。

```bash
bundle exec rails server -b '0.0.0.0' -p ${PORT:-3000}
```

## 4. 確認

ここまでで作成したファイルを確認します。

```bash
$ tree -L 1 -a
.
├── .bundle
├── .git
├── .gitattributes
├── .gitignore
├── .ruby-version
├── Dockerfile
├── Gemfile
├── Gemfile.lock
├── README.md
├── Rakefile
├── app
├── bin
├── config
├── config.ru
├── db
├── entrypoint.sh
├── lib
├── log
├── public
├── storage
├── test
├── tmp
└── vendor
```

# Herokuへのデプロイの準備

次にHerokuへのデプロイの準備をしていきます。

## 1. herokuコマンドのインストール

herokuコマンドがインストールされていない場合は、インストールします。

```bash
$ brew tap heroku/brew && brew install heroku
```

詳細は[公式サイト](https://devcenter.heroku.com/articles/heroku-cli#download-and-install)を参照してください。

## 2. heroku-cliにログイン

```bash
$ heroku login
```

## 3. Herokuでアプリを新規作成

```bash
$ heroku create my-app
```

## 4. DB用のアドオンを追加

### PostgreSQLの場合

無料のHobby Devを利用します。

```bash
$ heroku addons:create heroku-postgresql:hobby-dev
```

Herokuアプリ内に環境変数`DATABASE_URL`が自動で設定されます。

### MySQLの場合

無料のigniteを利用。

```bash
$ heroku addons:create cleardb:ignite
$ heroku config
CLEARDB_DATABASE_URL: mysql://[ユーザー名]:[パスワード]@[ホスト名]/[データベース名]?reconnect=true
```

`$ heroku config`を確認すると`CLEARDB_DATABASE_URL`という環境変数が追加されるので、その値を環境変数`DATABASE_URL`として設定します。

**※ Herokuでは（Rails 4.1 以上の場合）`config/database.yml`の内容は全て無視され、`DATABASE_URL`からDBの接続情報が読み込まれます。**（[参考1](https://devcenter.heroku.com/ja/articles/rails-database-connection-behavior#configuring-connections-in-rails-4-1)、[参考2](https://devcenter.heroku.com/ja/articles/ruby-support#build-behavior)）。

```bash
$ heroku config:add DATABASE_URL='mysql://[ユーザー名]:[パスワード]@[ホスト名]/[データベース名]?reconnect=true'
```

**※ `mysql2`を使っている場合は下のように設定しないとエラーになります。**

```bash
$ heroku config:add DATABASE_URL='mysql2://[ユーザー名]:[パスワード]@[ホスト名]/[データベース名]?reconnect=true'
```

## 5. 作成したファイルをコミット

```bash
$ bundle lock --add-platform x86_64-linux # MacOSを使用している場合
$ git add .
$ git commit -m "create rails app"
```

# Herokuへのデプロイと動作確認

## 1. アプリをHerokuにデプロイ

```bash
$ git push heroku main
```

## 2. 動作確認

```bash
$ heroku open
```

![](/images/21a3cf0aacee7c/image.jpg)

画像のように表示されればOKです🎉
ここまでやりきれてすごい！！！お疲れ様でした！

# 参考資料

https://devcenter.heroku.com/ja/categories/deploying-with-docker
https://qiita.com/kodai_0122/items/67c6d390f18698950440
