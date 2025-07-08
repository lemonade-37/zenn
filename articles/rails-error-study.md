---
title: "Ruby on Railsのエラー画面の読み方"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails"]
published: true
published_at: 2023-07-31 12:00
---
## 自己紹介
はじめまして、はると申します！
現在はスクールに通いながら学習をしており、学習開始から4ヶ月が経過しました。
　
## 概要
Ruby on Railsでアプリ開発中に出会うエラー画面の基本の読み方について、調べながら、勉強を兼ねてまとめたものをアウトプットしてみます。
（エラー例は、Ruby 3.1.3、Rails 7.0.5で作成した学習時間を記録するアプリで、記事のために意図的に起こしたものです）
　
## エラー画面の見方　例①NoMethodError
エラー画面の項目を、上から順に意味を確認していきます。

![](/images/rails-error-study/no-method-error.png)
　
#### エラータイトル
`NoMethodError in StudyTimesController#create`

- `NoMethodError`
    エラーの名前です。今回の場合、翻訳結果より「メソッドが無い」という意味です。

- `in StudyTimesController#create`
    どの場所でエラーが起こっているか記載されています。今回の場合、StudyTimesControllerのcreateアクションでエラーが起こっています。
　
#### エラーメッセージ
`undefined method 'sae' for #<StudyTime id: nil, user_id: 1, total_time: 0, created_at: nil, updated_at: nil, status: 0>`
　
- `undefined method 'sae'`
    今回の場合、翻訳結果より `sae`が未定義のメソッドであるという意味です。

- `for #<StudyTime id: nil, user_id: 1, total_time: 0, created_at: nil, updated_at: nil, status: 0>`
    「StudyTime」オブジェクトで`sae`メソッドを呼び出そうとしていることが示されています。
    格好内は、そのデータのカラムごとに値は何が入っているかが書かれています。
    id、created_at、updated_atの中身だけ、nilになっているため、この時点でうまくデータが保存できていないことがわかります。
　
#### Did you mean?
- `Did you mean?　save`
    「おそらくこちらをお探しですか？」という意味です。
    `sae`ではなく`save`では無いですか？（タイポしていませんか？）と言われています。
    この部分だけ見ればエラーの解消はできそうですが、全体を把握するために、一番下まで読んでいきます。
　
#### Rails.root: /Users/ディレクトリ名/ディレクトリ名/studying-tracker
このアプリケーションのルートディレクトリへのパスが示されています。
　
#### スタックトレース
`Application Trace | Framework Trace | Full Trace`
- スタックトレースとは？
    [引用記事](#参考引用記事)より、『プログラム実行のある時点において、そこに至るメソッド呼び出し元情報を遡るデータ。バックトレースともいう。』
    エラーの原因を突き止めるために表示してくれています。

- Application Traceとは？
    アプリ内のコードの中での、エラーが発生している箇所のスタックトレースを表示しています。
    ここに表示されているファイルからエラーの原因を探し始めると効率的になりそうです。

- Framework Traceとは？
    フレームワークかgemの中で、エラーが発生している箇所のスタックトレースを表示しています。
    アプリ上のコードだけが間違っている場合でも、Framework Traceが記載されているのは、 アプリはRailsフレームワーク上に構築されているためです。

- Full Traceとは？
    Application TraceとFramework Traceの両方を含んでいます。
　
#### Request
- `Parameters:{"utf8"=>"✓", "authenticity_token"=>"[FILTERED]", "commit"=>"学習を開始する"}`
    HTTPリクエストで送信されたパラメータが記載されます。
    今回の場合、学習を開始するためのフォームのsubmitボタンが押された際に送信された内容が記載されています。
    UFT-8のエンコードなどの部分は割愛します。
    `authenticity_token`はRailsにデフォルトで備わっているセキュリティ機能の１つです。

- `Toggle session dump`
    クリックするとトグルが開き、セッション情報が記載されています。

- `Toggle env dump`
    クリックするとトグルが開き、環境変数情報情報が記載されています。
　
#### Response
- `Headers:`
        HTTPレスポンスヘッダとは、HTTPリクエストに対してサーバーが返す反応のことです。
        今回の場合、エラーでHTTPリクエストが正しく送れていないため、レスポンスが無くこの部分が空欄になっている可能性があります。
　
#### エラー画面下のターミナル
デバッグを仕込んでいなくても、この部分でデバッグ時と同じように、インスタンス変数などの中身を確認したりすることができます。
　
## エラー例①の修正
StudyTimesController内の、sae→saveとタイポを修正
```
def create
  @study_time = current_user.study_times.build
  @study_time.save
  redirect_to study_times_path
end
```
　
## エラー画面の見方　例②Routing Error
Routesなどの、例①では表示されていなかった項目もあるため、もう1つ別のエラー例も見てみます。

![](/images/rails-error-study/routing-error.png)
　
#### エラータイトル
`Routing Error`
ルーティングのエラーが起こっています。
　
#### エラーメッセージ
`No route matches [GET] "/study_times/17"`

- `No route matches`
    翻訳結果より「あてはまるルート（ルーティング）が存在しない」という意味です。

- `[GET] "/study_times/17"`
    HTTPメソッドがGETで、パスが"/study_times/17"のリクエストに対して、適切なルーティングが見つからないことが記載されています。
　
#### Routes
`Routes match in priority from top to bottom`
翻訳結果より、「ルーティングでは優先順位が上から下へと適用される」という意味です。
　
## エラー例②の修正
StudyTimesController内の、study_time_path→study_times_pathとパス名を修正
```
def update
  @study_time = StudyTime.find(params[:id])
  elapsed_time = (Time.now - @study_time.created_at) / 60
  @study_time.total_time = elapsed_time.to_i
  @study_time.touch(:updated_at)
  @study_time.save
  redirect_to study_times_path
end
```

## まとめ
- エラーが起こった際は、エラータイトル、エラーメッセージ、Application Trace、Requestを重点的に見る。
- 見るだけで拒否反応と、どこから見たら良いかすらわかっていなかったエラー画面が、勉強したことで少し把握できるようになった。
- エラー画面以外にも、デバッグや、コンソールのログなどの情報を合わせて原因を探すことが重要なので、これからも問題解決の手段について学びを深めて行きたいと思います！
　
## 参考・引用記事
引用）[Ruby用語集 スタックトレース](https://docs.ruby-lang.org/ja/latest/doc/glossary.html#sa:~:text=%E3%82%AB%E3%83%AB%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97-,%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9,-stack%20trace)

- [スタックトレースとは - 意味をわかりやすく](https://e-words.jp/w/スタックトレース.html)
- [トレースとは - 意味をわかりやすく](https://e-words.jp/w/トレース.html)
- [スタックとは - 意味をわかりやすく](https://e-words.jp/w/スタック.html)
- [スタックトレース（stack trace）「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word13281.html)
- [【Ruby on Rails】エラー画面の見方とポイントを解説](https://ichigick.com/rails-error-view/)
- [【Rails】エラー解決まとめ（心構え、手順、個別の対処法まで）【初心者向け】](https://ichigick.com/rails-error-summary/)
- [Ruby のエラーメッセージを読み解く（初心者向け）その 2](https://qiita.com/scivola/items/77017693de371ab49667)
- [HTTPレスポンスヘッダとは - IT用語辞典 e-WordsIT用語辞典](https://e-words.jp/w/HTTPレスポンスヘッダ.html)
- [HTTPレスポンスヘッダ「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word1847.html)
