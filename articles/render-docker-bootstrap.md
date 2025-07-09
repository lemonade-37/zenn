---
title: "Docker・Rails7・BootstrapでRenderにデプロイできる環境構築"
emoji: "🍏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "Rails", "Bootstrap", "render", "環境構築" ]
published: true
published_at: 2024-06-25 12:00
---
## 自己紹介
はじめまして、はると申します。
駆け出しエンジニアとして働き始めて約4ヶ月弱が経過しました🐣


## 概要
下記環境で、[Render](https://render.com/)にデプロイまでできる環境構築についてまとめました。
実際に自分がデプロイまでしたアプリのコードはこちらです。
Stimulusの練習のために作成したミニアプリです🍎

https://github.com/satou-haruka-37/fruit_forest

### 環境
- Mac OS（Apple Silicon）
- Docker
- Ruby on Rails (Ruby 3.2.2、Rails 7.0.8)
- Bootstrap
- DBを使用しないミニアプリ作成のための環境だったため、DBはSQLite使用
- [Render](https://render.com/)にデプロイ


## 環境構築手順
### Docker環境構築
1. `touch Dockerfile`を実行します。
    Dockerfileの中身は以下のとおりです。
    ```Dockerfile:Dockerfile
    ARG RUBY_VERSION=ruby:3.2.2
    ARG NODE_VERSION=20

    FROM $RUBY_VERSION
    ARG RUBY_VERSION
    ARG NODE_VERSION
    ENV LANG C.UTF-8
    ENV TZ Asia/Tokyo
    RUN curl -sL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - \
    && wget --quiet -O - /tmp/pubkey.gpg https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update -qq \
    && apt-get install -y build-essential nodejs yarn
    RUN mkdir /app
    WORKDIR /app
    RUN gem install bundler
    COPY Gemfile /app/Gemfile
    COPY Gemfile.lock /app/Gemfile.lock
    COPY yarn.lock /app/yarn.lock
    RUN bundle install
    RUN yarn install
    COPY . /app

    COPY entrypoint.sh /usr/bin/
    RUN chmod +x /usr/bin/entrypoint.sh

    RUN bundle exec rails assets:precompile

    ENTRYPOINT ["entrypoint.sh"]
    CMD ["rails", "server", "-b", "0.0.0.0"]
    ```
    　　
2. `touch docker-compose.yml`を実行します。
    docker-compose.ymlの中身は以下のとおりです。
    ```yml:docker-compose.yml
    services:
      web:
        build: .
        command: bash -c "bundle && yarn && bundle exec rails db:prepare && yarn build && yarn build:css && rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
        tty: true
        stdin_open: true
        volumes:
          - .:/app
          - bundle_data:/usr/local/bundle:cached
          - node_modules:/app/node_modules
        environment:
          TZ: Asia/Tokyo
        ports:
          - "3000:3000"

    volumes:
      bundle_data:
      node_modules:
    ```
    　　
3. `touch entrypoint.sh`を実行します。
    entrypoint.shの中身は以下のとおりです。
    ```sh:entrypoint.sh
    #!/bin/bash

    set -e

    rm -f /app/tmp/pids/server.pid

    if [ "$RAILS_ENV" = "production" ]; then
    bundle exec rails assets:clobber
    bundle exec rails assets:precompile
    bundle exec rails db:migrate
    fi

    exec "$@"
    ```
    　　
4. `touch Gemfile Gemfile.lock yarn.lock`を実行します。
    Gemfileの中身は以下のとおりです。
    Gemfile.lockとyarn.lockは編集しません。
    ```Gemfile:Gemfile
    # frozen_string_literal: true

    source "https://rubygems.org"

    gem "rails", "~> 7.0.8"
    ```

### Rails newとセットアップ・Railsサーバー立ち上げ
1. 下記コマンドでRails newします。
    `Overwrite /app/Gemfile? (enter "h" for help) [Ynaqdhm]`と出てきた際には`y`にします。
    ```bash:bash
    (local)$ docker compose run --rm web rails new . --css=bootstrap --skip-jbuilder --skip-action-mailbox --skip-action-mailer --skip-test --skip-active-storage --skip-action-text
    ```
    　　
2. `docker compose build`を実行して再ビルドします。
　　
3. 下記コマンドでDBのセットアップをします。
    ```bash:bash
    (local)$ docker compose run --rm web rails db:create

    (local)$ docker compose run --rm web rails db:migrate
    ```
    　　
4. サーバーを立ち上げて確認します。
    `localhost:３０００/`にアクセスし、画像のような画面が表示されればOKです。
    ```bash:bash
    (local)$ docker compose up -d
    ```

    ![](/images/render-docker-bootstrap/image.png)
    　　

### デプロイ準備・デプロイ
1. 本番環境で`Blocked hosts: 〜〜.onrender.com`のようなエラーが出るのを防ぐため、
    `config/environments/development.rb内`に下記を追加します。
    ```ruby:development.rb
    Rails.application.configure do

      # 〜省略〜

      config.hosts.clear
    end
    ```
    　　
2. GitHubにpushして、Renderでデプロイすれば、完了です🎉
MASTERKEYだけRender上で設定を忘れずに、他は特に、
今回の環境においてはRender上で設定することはありません。

## まとめ
Renderは3ヶ月以上のDBのデータ保持には有料プランが必要ですが、DBを使用しない簡単なミニアプリであれば無料でデプロイできるのでおすすめです。
読んでいただきありがとうございました😊

　
## 参考記事
- [DockerでRailsの環境構築](https://qiita.com/fussy113/items/e9f7457ad4de74023ef6)
- [rails 本番環境のcompileエラーについて。[ActionView::Template::Error (The asset "application.css" is not present in the asset pipeline.):]](https://qiita.com/masarumosu/items/45c869a10cbf89d18075)
- [Rails 7.0でアセットパイプラインはどう変わるか](https://www.wantedly.com/companies/wantedly/post_articles/354873)
