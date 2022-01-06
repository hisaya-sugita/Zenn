---
title: "Dockerで構築したRuby on RailsアプリをHerokuにデプロイするまで"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails","Docker","Heroku"]
published: true
---

## 1. herokuコマンドのインストール

```bash
$ brew tap heroku/brew && brew install heroku
```

詳細は[公式サイト](https://devcenter.heroku.com/articles/heroku-cli#download-and-install)を参照してください。

## 2. heroku-cliにログイン

```bash
$ heroku login
```

## 3. Container Registryにログイン

```bash
$ heroku container:login
```

## 4. Herokuでアプリを新規作成

```bash
$ heroku create my-app
```

## 5. Container Registryにdocker imageをpush

```bash
$ heroku container:push web
```

## 6. DB用のアドオンを追加

### PostgreSQLの場合

今回は無料のHobby Devを利用します。

```bash
$ heroku addons:create heroku-postgresql:hobby-dev
```

### MySQLの場合

今回は無料のigniteを利用します。

```bash
$ heroku addons:create cleardb:ignite
```

## 7. 環境変数の登録

この辺りは各環境によって異なると思うので、よしなにしてください。

```bash
$ heroku config:add DB_NAME='' DB_USERNAME='' DB_PASSWORD='' DB_HOST='' DB_PORT=''
```

## 8. DBマイグレーション

まずは、heroku環境の中に入ります。

```bash
$ heroku run bash
```

heroku環境に入ることができたらDBのマイグレーションなどをします。

```bash
$ bundle install
$ bundle exec rails db:migrate
$ bundle exec rails assets:precompile
```

heroku環境から離脱します。

```bash
$ exit
```

## 9. 動作確認

```bash
$ heroku open
```
