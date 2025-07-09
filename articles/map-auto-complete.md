---
title: "【Rails7】Maps JavaScript APIで住所オートコンプリート入力の新規投稿機能を実装する"
emoji: "📍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails", "GoogleMapsAPI"]
published: true
published_at: 2024-07-16 12:00
---
## 自己紹介
はじめまして、はる（[@lemonade_37](https://twitter.com/lemonade_37)）と申します。
駆け出しエンジニアとして働き始めて約4ヶ月が経過しました🐣


## 概要
前回の記事から期間が空いてしまいましたが、 [【Rails7】Maps JavaScript APIを使って投稿を地図上にピンで表示する](https://zenn.dev/lemonade_37/articles/map-javascript-api) の続きとして、位置情報を含んだMapと連動できる新規投稿機能を実装していきます。


### 完成イメージ図

![](/images/map-auto-complete/image01.gif)

![](/images/map-auto-complete/image02.png)


### 環境
- Ruby: 3.2.2
- Rails: 7.0.8（esbuild）
- TailwindCSS
- DaisyUI


## 新規投稿機能の実装
前回の記事で、APIの導入、[geocoder](https://rubygems.org/gems/geocoder?locale=ja) の導入、投稿一覧ページの作成は既にされている状態からスタートします。

### コントローラー・ルーティング・ビューの作成

1. postsコントローラーに `new`, `create` アクションを追加します

    ```ruby:posts_controller.rb
    def new
      @post = Post.new
    end

    def create
      @post = Post.new(post_params)
      if @post.save
        redirect_to posts_path
      else
        render :new
      end
    end

    private

    def post_params
      params.require(:post).permit(:title, :address, :body)
    end
    ```
    　　
2. ルーティングにも `new`, `create` を追加します

    ```ruby:routes.rb
    resources :posts, only: %i[index new create]
    ```
    　　
3. フォーム部分のパーシャルのビューを作成します

    ```erb:views/posts/_form.html.erb
    <div class="card w-96 bg-base-100 shadow-xl">
      <div class="card-body">
        <%= form_with model: post, local: true do |f| %>
          <div>
            <%= f.label :title %>
            <%= f.text_field :title, placeholder: "スポット名を入力して下さい", class: "input input-bordered w-full max-w-xs" %>
          </div>

          <div>
            <%= f.label :address %>
            <%= f.text_field :address, placeholder: "住所を入力してください", class: "input input-bordered w-full max-w-xs" %>
          </div>

          <div>
            <%= f.label :body %>
            <%= f.text_field :body, placeholder: "おすすめポイントなど、一言", class: "input input-bordered w-full max-w-xs" %>
          </div>

          <%= f.submit class: "btn btn-info" %>
        <% end %>
      </div>
    </div>
    ```

    :::message
    card 部分は DaisyUI の class を使用してカードが浮いているような見た目にしています。
    :::
    　　
4. 新規投稿ページのビューファイルも作成し、先ほど作成したパーシャルを呼び出します

    ```erb:views/posts/new.html.erb
    <div class="flex justify-center items-center mt-20">
      <%= render 'form', { post: @post } %>
    </div>
    ```
    　　
5. この時点で、実在する住所を入力すれば、保存ができるようになっています

    :::message
    入力した住所から緯度と経度を入力してくれる動作については、`gem "geocoder"` の導入と、モデルの下記の記載のおかげでできるようになっています。（[前回の記事](https://zenn.dev/lemonade_37/articles/map-javascript-api)で設定済）

    ```ruby:post.rb
    class Post < ApplicationRecord
      geocoded_by :address
      after_validation :geocode
    end
    ```
    :::
　　
### Place Autocomplete を使った新規投稿時のオートコンプリート機能の実装
住所を手動で入力するのは大変なので、GoogleMap に登録されているスポットが自動で出てくるオートコンプリート機能をつけていきます。

1. Google Cloud Platform の API ライブラリで、Places API を検索して、有効にします

    ![](/images/map-auto-complete/image03.png)
    　　
2. Map の API Key の認証情報で、「API の制限」をしている場合は、有効にした Places API にもチェックを入れます（map JavaScript API と同じ API キーで実装可能です）

    ![](/images/map-auto-complete/image04.png)
    　　
3. 新規投稿フォームのビューに、`id` と `scriptタグ` を追加します

    ```diff_erb:views/posts/_form.html.erb
      <div class="card w-96 bg-base-100 shadow-xl">
        <div class="card-body">
          <%= form_with model: post, local: true do |f| %>
            <div>
              <%= f.label :title, for: "Title" %>
    +         <%= f.text_field :title, id: "Title", placeholder: "スポット名を入力して下さい", class: "input input-bordered w-full max-w-xs", autocomplete: "off" %>
            </div>

            <div>
              <%= f.label :address, for: "Address" %>
    +         <%= f.text_field :address, id: "Address", placeholder: "住所を入力してください", class: "input input-bordered w-full max-w-xs", autocomplete: "off" %>
            </div>

            <div>
              <%= f.label :body, for: "Body" %>
    +         <%= f.text_field :body, id: "Body", placeholder: "おすすめポイントなど、一言", class: "input input-bordered w-full max-w-xs", autocomplete: "off" %>
            </div>

            <%= f.submit class: "btn btn-info" %>
          <% end %>
        </div>
      </div>

    + <script src="https://maps.googleapis.com/maps/api/js?key=<%= ENV["GOOGLE_API_KEY"] %>&libraries=places&callback=initialize"></script>
    ```
    　　
4. JS ファイルを作成します

    ```diff_javascript:app/javascript/application.js
    // Entry point for the build script in your package.json
    import { Turbo } from "@hotwired/turbo-rails"
    Turbo.session.drive = false
    import "./controllers"

    + import './place_autocomplete.js'
    ```

    ```javascript:app/javascript/place_autocomplete.js
    // Place Autocomplete
    document.addEventListener('DOMContentLoaded', function () {
      const inputTitle = document.getElementById('Title');
      const inputAddress = document.getElementById('Address');

      // オートコンプリートのオプション
      const options = {
        types: ['establishment'],
        componentRestrictions: { country: 'JP' },
      };

      // オートコンプリート適用
      const autocompleteTitle = new google.maps.places.Autocomplete(inputTitle, options);
      const autocompleteAddress = new google.maps.places.Autocomplete(inputAddress, options);

      // タイトルのオートコンプリートが選択されたとき
      autocompleteTitle.addListener('place_changed', function() {
        const place = autocompleteTitle.getPlace();
        inputTitle.value = place.name;
        inputAddress.value = place.formatted_address;
      });

      // 住所のオートコンプリートが選択されたとき
      autocompleteAddress.addListener('place_changed', function() {
        const place = autocompleteAddress.getPlace();
        inputTitle.value = place.name;
        inputAddress.value = place.formatted_address;
      });
    });
    ```
    :::message
    `componentRestrictions: { country: 'JP' }`
    この設定を記載することで、オートコンプリートのサジェストを日本国内のスポットのみにできます。
    （都道府県ごとに制限することはできないようでした😵）

    また、使いやすさの面で、投稿タイトル・住所の欄のどちらで記載した際にもオートコンプリートが動くようにしました。
    :::
    　　
5. 存在しない住所で投稿できないバリデーション、投稿失敗した際のエラーメッセージを追加します（エラーメッセージを既に導入済みの方はバリデーションのみ追加してください）

    ```diff_ruby:post.rb
      class Post < ApplicationRecord
        geocoded_by :address
        after_validation :geocode

    +   # ついでに通常のバリデーションも追加
    +   validates :title, presence: true, length: { maximum: 255 }
    +   validates :address, presence: true
    +   validates :body, length: { maximum: 65_535 }
    +   validate :validate_address

    +   def validate_address
    +     geocoded = Geocoder.search(address)
    +     unless geocoded&.first&.coordinates.present?
    +       errors.add(:address, 'が存在しません') # 「住所が存在しません」と表示される
    +     end
    +   end
      end
    ```

    ```erb:views/shared/_error_messages.html.erb
    <% if object.errors.any? %>
      <div role="alert" class="alert alert-warning">
        <svg xmlns="http://www.w3.org/2000/svg" class="stroke-current shrink-0 h-6 w-6" fill="none" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" /></svg>
        <span>
          <% object.errors.full_messages.each do |msg| %>
            <li><%= msg %></li>
          <% end %>
        </span>
      </div>
    <% end %>
    ```
    ※ エラーの表現には [DaisyUI の Alert](https://daisyui.com/components/alert/) を使用しています。

    ```diff_erb:views/posts/_form.html.erb
      〜省略〜

      <%= form_with model: post, local: true do |f| %>
    +   <%= render 'shared/error_messages', object: f.object %>
        <div>
          <%= f.label :title, for: "Title" %>

      〜省略〜
    ```

    ```diff_ruby:posts_controller.rb
      def create
        @post = Post.new(post_params)
        if @post.save
          redirect_to posts_path
        else
    +     render :new, status: :unprocessable_entity
        end
      end
    ```

    :::message
    Rails7 特有の対応で、`status: :unprocessable_entity` を追加する必要があります。

    引用：[【Rails】Rails7.0でバリデーションのエラーメッセージが表示されない時の解決法](https://qiita.com/P-man_Brown/items/862503a638801fea01e7)
    :::
    　　
6. ここまでで、以下の機能が完成しました！🎉

    - 新規投稿機能
    - 住所オートコンプリート機能
    - 住所を入力すると緯度と経度も保存され、地図上に投稿が表示される機能（[前回の記事](https://zenn.dev/lemonade_37/articles/map-javascript-api)の範囲）

    　　
## まとめ
この機能を初めて実装した際はJavaScriptにも慣れておらず苦戦したので、自分のためにもまとめることができてよかったです！
ここまで読んでくださりありがとうございました！

　
## 参考・引用記事
### 引用記事
1）[【Rails】Rails7.0でバリデーションのエラーメッセージが表示されない時の解決法](https://qiita.com/P-man_Brown/items/862503a638801fea01e7)

### 参考記事
- [Place Autocomplete | Maps JavaScript API | Google for Developers](https://developers.google.com/maps/documentation/javascript/places-autocomplete?hl=ja)
- [【Rails6 / Google Map API】住所を投稿する際に予測変換させる（Autocomplete）](https://qiita.com/kaiyukun/items/08fac2a357852741f418)
- [Google Places Autocomplete を使って地名入力補助](http://kwski.net/api/988/#google_vignette)
