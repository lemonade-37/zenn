---
title: "【Rails7・Docker・Sidekiq】Sidekiq導入とメール自動送信のバックグラウンド処理の実装"
emoji: "🏃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails", "Sidekiq", "Docker"]
published: true
published_at: 2024-03-31 12:00
---
## 自己紹介
はじめまして、はると申します。
駆け出しエンジニアとして働き始めて約1ヶ月が経過しました🐣

## 概要
RailsアプリにSidekiq(redis)を導入し、メールの自動送信の機能を追加していきます。
この記事のために、メールを10分後に送信するだけのサンプルアプリを用意しました。
完成形のコードはこちらです。

https://github.com/satou-haruka-37/sidekiq_sample

### 環境
- Mac OS（Apple Silicon）
- Docker
- Ruby 3.2.3
- Rails 7.1.3
- Importmap
- Tailwindcss、daisyUI
- PostgreSQL

## バックグラウンド処理とは？
Railsアプリケーションでは、ユーザーが特定の動作を行った際に「投稿する」「画面を表示させる」などのアクションを実行します。
バックグラウンド処理とは例えば、「毎日特定の時間にメールを送信する」などの処理です。
ユーザーの行う操作とは独立して、サーバー側で非同期に実行されるタスクのことを言います。


## バックグラウンド処理を実装する方法について
バックグラウンド処理を実装する方法はさまざまな手段があります。
調べたもののみですが、それぞれの特徴についてまとめます✏️

#### cron
Linuxに標準で備わっているプログラムの一種で、設定すると処理を自動で定期的に実行することができます。[1)](#引用記事)
Railsアプリケーション内だけでなく、決まった時間にアプリ自体を立ち上げるなど、システムレベルでの実行も可能です。

#### whenever
cronジョブをRubyで扱いやすいように構文を提供してくれているRubyのgemです。
[gem whenever](https://github.com/javan/whenever)

#### Sidekiq
バックグラウンド処理を行うためのRubyのgemです。
同時に複数のバックグラウンドジョブを実行するなどの実装ができます。
定期的なジョブをスケジュールする拡張機能を使用するための、[sidekiq-cron](https://github.com/sidekiq-cron/sidekiq-cron)や、[sidekiq-scheduler](https://github.com/sidekiq-scheduler/sidekiq-scheduler)などの追加gemがあります。
[gem sidekiq](https://github.com/sidekiq/sidekiq)

#### Redis
多目的に利用されるインメモリデータストアです。
インメモリデータストアは、すべてのデータをメモリ上に展開し、データの読み込みや追加、変更、削除をすべてメモリ上で完結させることで、従来型の数百倍から数万倍も高速に処理を実行することができるデータベースです。[2)](#引用記事)
今回はセッション管理として利用しますが、他にも様々な用途で使用されます。

#### ActiveJob
Rails4.2からrailsのデフォルトで導入された、これまでSidekiqなどのGemを使って行っていたバックグラウンドジョブ系を扱える機能です。
[ActiveJobの基礎](https://railsguides.jp/active_job_basics.html)


## 使用技術の選び方について
バックグラウンドでどんな処理を実行したいかという目的によって、上記の技術を必要に応じて組み合わせて使用します。

今回は、アプリサーバーは自動で落ちずに常に起動している状態で、決まったタイミングでメールを自動送信するという機能のみなので、
**ActiveJob＋Sidekiq（Redis）**　を使用することにしました。
　　
ActiveJobを介してSidekiqを使うか、直接Sidekiqを使用するかに関しては、調べてみたところさまざまな価値観があるようでした。

私は下記の記事を参考に、この組み合わせにしました。
（他にも参考にした記事は[参考記事](#参考記事)参照）

https://qiita.com/seiyatakahashi/items/cb9ae73e5ba3020f4a89#activejob%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%9B%E3%81%9Asidekiq%E3%82%92%E5%88%A9%E7%94%A8%E3%81%99%E3%82%8B


## 実装開始！🕰
それではここから、メールを送信するボタンを押すとエンキューされ、10分後に自動的にメールが送信される機能を実装していきます。
　　
基本的なメール送信機能はすでに実装されている状態からスタートします。
今回は、`gem 'letter_opener_web'`を使用し、ローカルのみでメール送信の動作を行えるようにしています。
この時点でのコードについて確認したい方は以下を参照ください。

https://github.com/satou-haruka-37/sidekiq_sample/tree/3af4a9a095d7c4f2f46c616ffb8ec366a40efc43

### Sidekiq・Redisの導入
1. docker-compose.ymlにredisを追加します。
    ```yml:docker-compose.yml
    redis:
      image: "redis:7.0-alpine"
      volumes:
        - redis_volume:/data
      command: redis-server --appendonly yes
      ports:
        - "6379:6379"
    ```
    `volumes:`にも`redis_volume:`を追加します。
    ```yml:docker-compose.yml
    volumes:
      postgres_volume:
      redis_volume:
    ```

    - ここで[DokerHubのイメージ](https://hub.docker.com/_/redis)を使用して、Redisのコンテナを作成します。

    - Dockerのボリュームについての説明は今回は割愛します。（[公式ドキュメント](https://matsuand.github.io/docs.docker.jp.onthefly/storage/volumes/)を参照ください）

    - ボリュームを設定し、`command: redis-server --appendonly yes`と記載しAOF Redis を使用することで、
    10分後送信予定のメールのキューがある状態でDockerが停止してしまっても、
    実行予定で溜まっていたキューが削除されずに、
    Dockerを再度立ち上げるとキューの続きから実行することができます。
　　
2. docker-compose.ymlにsidekiqを追加します。
    ```yml:docker-compose.yml
    sidekiq:
      build: .
      command: bundle exec sidekiq
      volumes:
        - .:/app
      depends_on:
        - db
        - redis
    ```
    - `depends_on:`でdbコンテナとredisコンテナがSidekiqサービスより先に起動するように指定します。
    　　

3. `docker compose build`コマンドで再ビルドします。
    　　
4. Gemfileに下記を記載し、`docker compose up`してから、`docker compose exec web bundle install`実行します。
    ```Gemfile:Gemfile
    # Background Job
    gem 'sidekiq'
    gem 'sidekiq-scheduler'
    ```
    　　
5. config/application.rbに下記を追加します。
    ```ruby:application.rb
    class Application < Rails::Application
      # 〜〜省略〜〜
      config.active_job.queue_adapter = :sidekiq
      # 〜〜省略〜〜
    end
    ```
    　　
6. config/initializers/sidekiq.rbを作成し、下記のように記載します。
    ```ruby:sidekiq.rb
    Sidekiq.configure_server do |config|
      config.redis = { url: 'redis://redis:6379' }
    end

    Sidekiq.configure_client do |config|
      config.redis = { url: 'redis://redis:6379' }
    end
    ```
    　　
7. config/sidekiq.ymlを作成し、下記のように記載します。
    ```yml:sidekiq.yml
    :concurrency: 3
    :queues:
      - default
    ```
    - `:concurrency:`は同時並行で実施できるジョブの数の上限を指定しているので、任意の数値でOKです。
    　　
8. `docker compose exec web bundle exec sidekiq`を実行して、下記のように起動されれば導入完了です🦶
    ```ターミナル
                   m,
                   `$b
              .ss,  $$:         .,d$
              `$$P,d$P'    .,md$P"'
               ,$$$$$b/md$$$P^'
             .d$$$$$$/$$$P'
             $$^' `"/$$$'       ____  _     _      _    _
             $:    ',$$:       / ___|(_) __| | ___| | _(_) __ _
             `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
                    $$:         ___) | | (_| |  __/   <| | (_| |
                    $$         |____/|_|\__,_|\___|_|\_\_|\__, |
                  .d$$                                       |_|


    〜〜省略〜〜

    INFO: Starting processing, hit Ctrl-C to stop
    ```
    　　
### Sidekiqダッシュボードの設定
1. config/routes.rbに下記を追加します。
    ```ruby:routes.rb
    require 'sidekiq/web'

    Rails.application.routes.draw do
      mount Sidekiq::Web, at: '/sidekiq'
    end
    ```
    　　
2. Dockerを立ち上げ直し、[http://localhost:3000/sidekiq/](http://localhost:3000/sidekiq/)　にアクセスして、下記の画面が表示されればダッシュボードの設定が完了です。
    ![](/images/rails-docker-sidekiq/image01.png)
    　　
3. `※任意` ダッシュボードには一般ユーザーはアクセスできないようにBasic認証をつけるのが望ましいです。
    ```ruby:routes.rb
    require 'sidekiq/web'

    Rails.application.routes.draw do
      Sidekiq::Web.use(Rack::Auth::Basic) do |user_id, password|
        [user_id, password] == [ENV['SIDEKIQ_BASIC_ID'], ENV['SIDEKIQ_BASIC_PASSWORD']]
      end
      mount Sidekiq::Web, at: '/sidekiq'
    end
    ```
    ```.env:.env
    SIDEKIQ_BASIC_ID=hogehoge
    SIDEKIQ_BASIC_PASSWORD=fugafuga
    ```
    Basic認証が追加できると以下のように、パスワードを入れないとアクセスできなくなります。
    ![](/images/rails-docker-sidekiq/image02.png)
    　　
### 10分後に自動送信されるメールのActiveJob作成
ここからは、メールの定期的な自動送信機能のジョブを作成します。

1. ActiveJobのファイルを作成します。
    `docker compose exec web rails g job SendEmail`を実行し、app/jobs/send_email_job.rbを作成します。
    ジョブの中でMail送信を実行するようにします。
    ```ruby:send_email_job.rb
    class SendEmailJob < ApplicationJob
      queue_as :default

      def perform(email)
        SampleEmailMailer.sample_email(email).deliver_now
      end
    end
    ```
    　　
2. コントローラーでJobを呼び出します。
    現在時刻から10分後に実行するキューを追加する形とします。
    ```diff_ruby:controllers/static_pages_controller.rb
    class StaticPagesController < ApplicationController
      def new ; end

      def create
        email = params[:email]
    +   SendEmailJob.set(wait_until: Time.zone.now + 10.minutes).perform_later(email) #10分後に実行するジョブをキューに追加
        redirect_to new_static_page_path, notice: 'メールを送信のキューを追加しました'
      end
    end
    ```
    　　
3. Railsコンソールを立ち上げて、ジョブ自体が実行できるか確認してみます。
    ```c:ターミナル
    % docker compose exec web rails c
    Loading development environment (Rails 7.1.3.2)
    irb(main):001> SendEmailJob.perform_now("a@example.com")
    ```
    　　
4. ジョブが実行され、メールが届いていれば、ジョブ自体の実行はできています。
![](/images/rails-docker-sidekiq/image03.png)

### 10分後に自動送信されるメールの動作確認
では、実際アプリ画面でメールを送信し、10分後に実行されるか確認します。

1. まずDockerを立ち上げた後忘れずに、Sidekiqを立ち上げます。
    （これを忘れているとキューが実行されないので注意⚠️）
    ```c:ターミナル
    docker compose exec web bundle exec sidekiq
    ```
    　　
2. アプリからメールを送信してみます。
![](/images/rails-docker-sidekiq/image04.gif)
　　
3. Sidekiqのダッシュボードにアクセスし、`予定`タブに予定されたジョブが追加されていることを確認します。
![](/images/rails-docker-sidekiq/image05.png)

4. 10分経過後にメールが自動送信されていることを確認できたら、実装完了です！！🎉
![](/images/rails-docker-sidekiq/image06.png)


### 指定した日時に自動送信されるように設定
10分後に送信する以外にも、「毎月の月初にメールを送信する」などの設定をすることもできます。

1. config/sidekiq.ymlに下記の`:scheduler:`を追加します。
    ```yml:sidekiq.yml
    :concurrency: 3
    :queues:
      - default
    :scheduler:
      :schedule:
        send_email_job:
          cron: "0 12 1 * *"
          class: SendEmailJob
          queue: default
    ```
    　　
2. sidekiqを立ち上げ直した際に、ターミナルに下記が表示されていれば、指定した日時にジョブを実行するスケジュールが設定されています🎉
    ```:ターミナル
    2024-03-31T12:46:26.925Z pid=67 tid=2vv INFO: Scheduling send_email_job {"cron"=>"0 12 1 * *", "class"=>"SendEmailJob", "queue"=>"default"}
    ```
    　　

## さいごに
業務の中でメールの自動送信の機能を作成する機会があり、Sidekiqを初めて使用しましたが、
これまで理解できていなかった部分だったため、大変勉強になりました！🕰️
まだまだ理解しきれていない部分も多いため、引き続き勉強していきます🙇

　
## 参考・引用記事
#### 引用記事
1）[cronとは。crontabで定期実行を自動化する方法を解説](https://www.miraiserver.ne.jp/column/about_cron/#:~:text=cron%E3%81%A8%E3%81%AF%E3%80%81Linux%E3%81%AB,%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E3%81%A8%E4%BE%BF%E5%88%A9%E3%81%A7%E3%81%99%E3%80%82)
2）[e-Words インメモリデータベース](https://e-words.jp/w/%E3%82%A4%E3%83%B3%E3%83%A1%E3%83%A2%E3%83%AA%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9.html)

#### 参考記事
- [Wikipedia バックグラウンドプロセス](https://ja.wikipedia.org/wiki/%E3%83%90%E3%83%83%E3%82%AF%E3%82%B0%E3%83%A9%E3%82%A6%E3%83%B3%E3%83%89%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9)
- [【個人開発】スクール生活をより豊かにするためのアプリを開発しました🤖](https://qiita.com/saki_y/items/d6e28d06462e1a6fe919)
- [インメモリDBとは](https://qiita.com/miyuki_samitani/items/36699533f08221ff5bcf)
- [Rails: SidekiqはActive Jobを経由せずに直接使おう（翻訳）](https://techracho.bpsinc.jp/hachi8833/2023_03_06/127675)
- [ActiveJob はまだちょっと使うには早いかも](https://blog.willnet.in/entry/2015/07/24/171034)
- [redisをdockerで起動してみた。](https://qiita.com/mojapico/items/497fd9483cbc0f1bfe11)
- [docker-composeでredis環境をつくる](https://qiita.com/uggds/items/5e4f8fee180d77c06ee1)
- [RedisをDockerで実行する方法（＋それをおすすめする理由）](https://kinsta.com/jp/blog/redis-docker/)
- [Redis の永続性](https://redis.io/docs/management/persistence/)
- [【Rails7】Redis+Sidekiq+Active Jobで指定した日時に非同期処理を行う](https://zenn.dev/yoiyoicho/articles/03863702867eb0)
- [Sidekiqの管理画面をアクセス制限する方法3選](https://zenn.dev/n04h/articles/sidekiq-auth)
- [sidekiq + active jobで予定されたjobが実行状態にならない](https://teratail.com/questions/326549)
