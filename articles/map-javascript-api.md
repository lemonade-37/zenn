---
title: "【Rails7】Maps JavaScript APIを使って投稿を地図上にピンで表示する"
emoji: "🗺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails", "GoogleMapsAPI"]
published: true
published_at: 2023-12-15 12:00
---
## 自己紹介
はじめまして、はると申します。現在はスクールに通いながら学習をしています。
今回、[スクール内のアドベントカレンダー](https://qiita.com/advent-calendar/2023/runteq)に参加し、**『初めて学んだ技術』** というテーマで記事を書きました🐥

## 概要
Google MapのAPIを利用し、地図上に投稿の一覧をピンで表示させます。
（新規投稿を作成する機能の実装は含みません）
- Maps JavaScript API
- Geocoding API

### 環境
- Ruby: 3.2.2
- Rails: 7.0.8
- esbuild


## APIキーの準備
初めてGoogle Cloud Platformを使う想定で記載します。すでに登録済みの方は[使用するAPIの有効化](#使用するapiの有効化)に進んでください。

### Google Cloud Platformにログイン
[Google Cloud Platform](https://cloud.google.com/?hl=ja)にアクセスし、`無料で利用開始`から案内に沿って初期設定を行います。
Map javaScript APIは無料トライアルで使えるAPIには含まれていないため、この時に、クレジットカード情報も登録しておくと良いです。

![](/images/map-javascript-api/image01.png)

### お支払い方法の設定
ログインができたら、Google Cloudのコンソールに遷移されます。
無料トライアルの状態であることがページ上部に表示されているかと思います。
`有効化`のボタンから、課金の有効化をします。
（現在は課金にしても、無料トライアルの残りクレジットを使い切るまで or 3ヶ月間は、課金が開始されません）

これをしないと、APIキーが使えないエラーが後で起こるので、設定しておきましょう。

### 使用するAPIの有効化
コンソール画面左上のメニューを開き、`APIとサービス`内の、`ライブラリ`に進みます。
![](/images/map-javascript-api/image02.png)

このような画面に遷移するので、検索フォームから、`Maps JavaScript API`と、`Geocoding API`を検索します。
![](/images/map-javascript-api/image03.png)

2つのAPIを有効化します。
![](/images/map-javascript-api/image04.png)
![](/images/map-javascript-api/image05.png)

有効化した際に表示される**APIキーを控えておきます。（後でいつでも確認できます）**


### APIの制限をつける（任意）
念の為、セキュリティ上、不正利用された際などに必要以上にAPIを使用されないよう、
選択したAPIしか使えなくなる制限をかけておきます。

APIとサービスにアクセスした状態で、サイドバーから`認証情報`にアクセスします。
![](/images/map-javascript-api/image06.png)

先ほど発行されたAPIキーが表示されているため、クリックして詳細ページへ移動します。
![](/images/map-javascript-api/image07.png)

APIの制限で、`キーを制限`を選択し、`Maps JavaScript API`と`Geocoding API`にチェックを入れて設定を保存します。
![](/images/map-javascript-api/image08.png)
![](/images/map-javascript-api/image09.png)


## APIの導入
`rails new`してから初めてAPIを導入する想定で記載します。すでに.env等の設定が終わっている方は[APIキーの設定・表示確認](#apiキーの設定・表示確認)に進んでください。

### APIキーを扱うための準備（重要⭕️）
1. Gemfileに下記を追加し、`$ bundle install`を実行します。
    ```ruby:Gemfile
    gem 'dotenv-rails'
    ```
    　
2. **.gitignoreファイルに下記を追加**
    ```ruby:.gitignore
    /.env
    ```
    :::message alert
    この作業を忘れると、GitHubにAPIキー（**機密情報**）を載せてしまう可能性があるので、先に済ませておきましょう！
    :::


3. `$ touch .env`で.envファイルを作成します。
　
### APIキーの設定・表示確認
1. .envファイル内に取得したAPIキーを記載します。
    ```ruby:.env
    GOOGLE_MAPS_API_KEY = "取得したAPIキーをここに記載"
    ```
    　
2. 表示確認のため、試しにどこかのビューに下記を記載してみます。（任意）
    ```html:hoge.html.erb
    <!-- マップを表示する要素 -->
    <div id="map" style="height: 600px;"></div>

    <script>
      // 地図を初期化する関数
      function initMap() {
        // 地図のオプション
        const mapOptions = {
          center: { lat: 35.6803997, lng: 139.7690174 }, // 地図の初期表示位置（東京）
          zoom: 10 // ズームレベル
        };

        // 地図を指定した要素に表示
        const map = new google.maps.Map(document.getElementById('map'), mapOptions);
      }
    </script>
    <script src="https://maps.googleapis.com/maps/api/js?key=<%= ENV["GOOGLE_MAPS_API_KEY"] %>&callback=initMap"></script>
    ```
    　
4. 地図が表示されていればAPIの導入は成功です🎉（任意）
    ![](/images/map-javascript-api/image10.png)



## 投稿に関する機能作成
1. Postモデル、postsテーブルを作成します。
    `$ rails g model Post title:string addless:text latitude:float longitude:float`を実行します。
    投稿の中に文章など含めたいものがあれば、一緒にカラムを追加して構いません。今回は記事のため最低限、`title`のみを入れています。
    　
2. 生成されたマイグレーションファイルを編集します。
    ```ruby:xxxxxxxxxxxxxx_create_posts.rb
    class CreatePosts < ActiveRecord::Migration[7.0]
      def change
        create_table :posts do |t|
          t.string :title, null: false
          t.text :address, null: false
          t.float :latitude, null: false
          t.float :longitude, null: false

          t.timestamps
        end
      end
    end
    ```
    　
3. `$ rails db:migrate`を実行します。
    　
4. コントローラー、indexアクションを作成します。
    `$ rails g controller Posts`を実行します。
    ```ruby:posts_controller.rb
    class PostsController < ApplicationController
      def index
        @posts = Post.all
      end
    end
    ```

    　
5. ルーティングに下記を追加します。
    ```ruby:routes.rb
    resources :posts, only: %i[index]
    ```
    　
6. Gemfileに下記を追加し、`$ bundle install`を実行します。
    ```ruby:Gemfile
    gem "geocoder"
    ```
    　
7. `$ rails g geocoder:config`を実行します。
    　
8. config/initializers/geocoder.rbが生成されるので、以下のように編集します。
    ```ruby:geocoder.rb
    Geocoder.configure(
      # Geocoding options
      # timeout: 3,                 # geocoding service timeout (secs)
      lookup: :google,         # name of geocoding service (symbol)
      # ip_lookup: :ipinfo_io,      # name of IP address geocoding service (symbol)
      # language: :en,              # ISO-639 language code
      use_https: true,           # use HTTPS for lookup requests? (if supported)
      # http_proxy: nil,            # HTTP proxy server (user:pass@host:port)
      # https_proxy: nil,           # HTTPS proxy server (user:pass@host:port)
      api_key: ENV['GOOGLE_MAPS_API_KEY'],               # API key for geocoding service
      # cache: nil,                 # cache object (must respond to #[], #[]=, and #del)

      # Exceptions that should not be rescued by default
      # (if you want to implement custom error handling);
      # supports SocketError and Timeout::Error
      # always_raise: [],

      # Calculation options
      # units: :mi,                 # :km for kilometers or :mi for miles
      # distances: :linear          # :spherical or :linear

      # Cache configuration
      # cache_options: {
      #   expiration: 2.days,
      #   prefix: 'geocoder:'
      # }
    )
    ```
    　
9. Postモデルのファイルを編集します。
    ```ruby:post.rb
    class Post < ApplicationRecord
      geocoded_by :address
      after_validation :geocode
    end
    ```
    　
10. views/posts/index.html.erbを作成します。
    ```html:index.html.erb
    <div id="map" style="height: 600px;"></div>

    <script>
      function initMap() {
        // 地図要素を取得する（マーカーを表示させるために必要）
        const mapElement = document.getElementById('map');

        const mapOptions = {
          center: { lat: 35.6803997, lng: 139.7690174 },
          zoom: 10
        };

        const map = new google.maps.Map(mapElement, mapOptions);

        // マーカーを追加（Postの情報からマーカーを追加する）
        <% @posts.each do |post| %>
          new google.maps.Marker({
            position: {lat: <%= post.latitude %>, lng: <%= post.longitude %>}, 
            map: map,
            title: '<%= j post.title %>'
          });
        <% end %>
      }
    </script>
    <script src="https://maps.googleapis.com/maps/api/js?key=<%= ENV["GOOGLE_MAPS_API_KEY"] %>&callback=initMap"></script>
    ```



## マップ上に投稿がピンで表示されているか動作確認
今回は新規投稿を作成する機能までは実装しないため、
seedデータを使って、地図上に投稿が表示されるかの動作確認をします。

1. Gemfileに下記を追加し、`$ bundle install`を実行します。
    ```ruby:Gemfile
    gem 'faker'
    ```
    　
2. db/seeds.rbを編集します。
    Fakerは架空の住所を生成するためのツールのため、実在する緯度・経度にならない可能性があるので、`address`ではなく緯度と経度に直接Fakerを使用しています。
    ```ruby:seeds.rb
    10.times do |n|
      latitude = Faker::Address.latitude.to_f
      longitude = Faker::Address.longitude.to_f
      Post.create!(
        title: "Example Title #{n + 1}",
        address: "Example Address #{n + 1}",
        latitude: latitude,
        longitude: longitude,
      )
    end
    ```
    　
3. `$ rails db:seed`を実行します。
    　
4. `/posts`にアクセスして、地図上にpostsテーブルに登録されている位置情報のピンが表示されていれば、成功です！🥳🎊
    Fakerで作ったpostの位置情報は、世界の中からランダムに作られるため、ズームレベルを遠くしてみると、ピンが表示されていることを確認できます。
    ![](/images/map-javascript-api/image11.png)


## まとめ
投稿データをMap上にピンで表示する方法をまとめました。
アプリ開発を始めたばかりで、APIを触るのも、Map JavaScript APIも初めてでしたが、表示することができて良かったです。
この後は、ユーザーが投稿したデータを表示できるようにするために、投稿の作成機能も実装していきたいと思います✊🏻


## 参考記事
- [API Keyを発行する手順 for Google Cloud API](https://zenn.dev/tmitsuoka0423/articles/get-gcp-api-key)
- [API key等をgithubで公開しない方法(rails,heroku)](https://qiita.com/uma0317/items/e142661c004f68d858a5)
- [【Rails6 / Google Map API】初学者向け！Ruby on Railsで簡単にGoogle Map APIの導入する](https://qiita.com/nagase_toya/items/e49977efb686ed05eadb)
- [【Rails7】住所を入力しただけで地図(Googleマップ)が自動表示されるようにする](https://plog.kobacchi.com/rails-auto-generate-googlemaps-from-address/#index_id8)
- [[Rails] Google JavaScript APIを使い投稿機能を実装する](https://zenn.dev/yuna0960740/articles/98531e13665815)
