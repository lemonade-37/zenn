---
title: "Docker・Rails7・TailwindCSS・DaisyUIでFly.ioにデプロイできる環境構築"
emoji: "🌼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "Docker", "Flyio", "TailwindCSS", "daisyUI"]
published: true
published_at: 2024-06-25 12:00
---
## 自己紹介
はじめまして、はる（[@lemonade_37](https://twitter.com/lemonade_37)）と申します。
駆け出しエンジニアとして働き始めて約4ヶ月弱が経過しました🐣


## 概要
下記環境で、[Fly.io](https://fly.io/)にデプロイまでできる環境構築についてまとめました。

### 環境
- Mac OS（Apple Silicon）
- Docker
- Ruby on Rails (Ruby 3.2.3、Rails 7.1.3)
- Tailwind
- DaisyUI
- PostgreSQL（Fly Postgres）
- [Fly.io](https://fly.io/)にデプロイ


## 環境構築手順
### Docker環境構築
1. `touch Dockerfile`を実行します。
    Dockerfileの中身は以下のとおりです。
    ```Dockerfile:Dockerfile
    ARG RUBY_VERSION=ruby:3.2.3
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
      db:
        image: postgres:latest
        platform: linux/amd64
        environment:
          TZ: Asia/Tokyo
          POSTGRES_PASSWORD: password
        volumes:
          - postgres_data:/var/lib/postgresql/data
        ports:
          - 5432:5432
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
        depends_on:
          - db

    volumes:
      postgres_data:
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

    gem "rails", "~> 7.1.3"
    ```

### Rails newとDBセットアップ・Railsサーバー立ち上げ
1. 下記コマンドでRails newします。
    `Overwrite /app/Gemfile? (enter "h" for help) [Ynaqdhm]`と出てきた際には`y`にします。
    ```bash:bash
    (local)$ docker compose run --rm web rails new .  --database=postgresql --css=tailwind --javascript=esbuild --skip-action-mailbox --skip-action-mailer --skip-test --skip-active-storage --skip-action-text
    ```
    　　
2. `docker compose build`を実行して再ビルドします。
　　
3. config/database.ymlに以下を追加します。
    ```diff_ruby:database.yml
    development:
      <<: *default
      database: app_development
    + username: postgres
    + password: password
    + host: db
    + port: 5432

    test:
      <<: *default
      database: app_test
    + username: postgres
    + password: password
    + host: db
    + port: 5432
    ```
    　　
4. 下記コマンドでDBのセットアップをします。
    ```bash:bash
    (local)$ docker compose run --rm web rails db:create

    (local)$ docker compose run --rm web rails db:migrate
    ```
    　　
4. サーバーを立ち上げて確認します。
    `localhost:３０００/`にアクセスし、画像のような画面が表示されればOKです。
    ```bash:bash
    (local)$ docker compose up -d
    ```

    ![](/images/flyio-docker-rails-tailwind-daisyui/image01.jpg)
    　　

### DaisyUIのインストール
1. 下記コマンドを実行します。
    ```bash:bash
    (local)$ docker compose exec web yarn add daisyui
    ```
    　　
2. tailwind.config.jsに以下を追加します。
    ```diff_javascript:tailwind.config.js
    module.exports = {
      content: [
        './app/views/**/*.html.erb',
        './app/helpers/**/*.rb',
        './app/assets/stylesheets/**/*.css',
        './app/javascript/**/*.js'
      ],
    + plugins: [require("daisyui")],
    }
    ```

### デプロイ準備・デプロイ
1. 本番環境で`Blocked hosts: 〜〜.onrender.com`のようなエラーが出るのを防ぐため、
    `config/environments/development.rb内`に下記を追加します。
    ```ruby:development.rb
    Rails.application.configure do

      # 〜省略〜

      config.hosts.clear
    end
    ```
    　　
2. あらかじめFly.ioのアカウントを作っておくとスムーズです。（GitHubログインが楽です）
    クレジットカードの登録は必須です。(現在、従量課金で5$以下は無料ですが、詳しくは公式でご確認ください。)
    また、Fly.ioを初めて使うorコマンドをインストールしてない場合は、flyのためのコマンドのインストールをします。

    ```bash:bash
    (local)$ brew install flyctl
    ```
    　　
3. アプリのフォルダ内にいる状態で下記コマンドを実行します。
    ```bash:bash
    (local)$ fly auth login
    ```
    すると、ブラウザが自動で開くため、画面内の「Continue as 〇〇〜」をクリックします。
    その後、ターミナルに戻り、`successfully logged in as 〇〇`と表示されていればOKです。
    　　
4. 次に下記コマンドを実行します。
    ```bash:bash
    (local)$ fly launch
    ```
    ターミナルで、`? Do you want to tweak these settings before proceeding? (y/N)`と出てくるので、`y`を選択します。
    すると、ブラウザ上の設定ページに自動で飛ばされます。このような画面です。

    ![](/images/flyio-docker-rails-tailwind-daisyui/image02.png)

    :::message alert
    **課金されないために**
    この設定画面内で、**Databaseのconfigurationの設定を`Development〜`に変更します。**
    その他、（アプリ名、DB名は任意で設定してください。）
    　　
5. 設定後、`Confirm Setting`をクリックします。
    すると、ターミナルが動き出し、デプロイの準備が行われます。
    `Once ready: run 'fly deploy' to deploy your Rails app.`が表示されれば完了し、デプロイの準備OKです。
    　　
6. 次に下記コマンドを実行してデプロイします。
    ```bash:bash
    (local)$ fly deploy
    ```
    `Visit your newly deployed app at https://アプリ名.fly.dev/`と表示されるためURLにアクセスしRailsの初期画面が表示されていればデプロイ完了です🎉


## まとめ
以前Fly.ioでDockerを使ったデプロイをするのに苦戦したことがあったのですが、この環境であれば無事にデプロイまで行うことができました！
読んでいただきありがとうございました😊

　
## 参考記事
- [Launch a demo app · Fly Docs](https://fly.io/docs/getting-started/launch-demo/)
- [Getting Started · Fly Docs](https://fly.io/docs/rails/getting-started/)
