---
title: "【Rails】GMOあおぞらネット銀行指定形式CSVエクスポートの実装"
emoji: "📄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails", "CSV"]
published: true
published_at: 2024-03-31 12:00
---
## 自己紹介
はじめまして、はる（[@lemonade_37](https://twitter.com/lemonade_37)）と申します。
駆け出しエンジニアとして働き始めて約1ヶ月が経過しました🐣

## 概要
業務内で、GMOあおぞらネット銀行指定形式のCSVファイルをエクスポートする機能を実装する必要がありましたが、なかなか想定する形式で出力できず苦労したためまとめました✏️

### 環境
- Ruby 3.2.3
- Rails 7.1.3
- Importmap
- Tailwindcss、daisyUI
- PostgreSQL


## エクスポート・インポートとは？
改めて言葉の定義を確認しておきます。
#### エクスポートとは
> あるソフトウェアで作成・編集したデータを他のソフトが読み込める形式に変換したり、そのような形式でファイルに保存することを指す。[1）](#引用記事)
#### インポートとは
> あるソフトウェアが、他のソフトで作成されたデータやファイルを読み込んで利用できるようにすることをこのように呼ぶ。[2）](#引用記事)


## CSVエクスポート機能の必要性
会社から複数名のお客様にお金の振り込みを行うなど、GMOあおぞらネット銀行の企業用口座から、複数口座に一括振り込みを行う際に、[「一括振込（WEBアップロード） ご利用ガイド」](https://gmo-aozora.com/support/guide.html)からダウンロードできるPDFで説明されている書式に沿ったCSVファイルが必要になります。


## 実装方法
Railsのアプリで、下記のようなテーブル設計があるとして実装します。ユーザーごとに決まった金額を、お店側からユーザーへ一括で振り込む状況を想定しています。
$\tiny{命名のセンスが無いのは多めに見てください…！}$

`account_informations`テーブルは口座情報のテーブルですが、銀行コードや支店コードなど、全て **`string`（文字列）型** を使用しています！
これは、電話番号などでも同じですが、データベースの性質上、`integer`型だと先頭の0が省略されてしまうためです。

![](/images/rails-gmo-csv/image01.png)


サンプルコードはこちらです。

https://github.com/satou-haruka-37/csv_sample

### 1. コントローラーのアクション定義
RubyのCSVライブラリを使用できるようにします。（追加後はRailsサーバーの再起動が必要です。）
```diff_ruby:controllers/static_pages_controller.rb
class StaticPagesController < ApplicationController
+ require 'csv'
```
　　
`export`アクションを定義します。
```ruby:controllers/static_pages_controller.rb
def export
  @account_informations = AccountInformation.all

  respond_to do |format|
    format.csv { send_data generate_csv(@account_informations), filename: "CSVサンプル.csv" }
  end
end
```
:::message
`filename`では、エクスポート時のCSVファイルのファイル名を指定することができます。
:::

　　
`export`アクション内で呼び出す`generate_csv`メソッドを定義します。
```ruby:controllers/static_pages_controller.rb
private

# GMOあおぞらネット銀行指定形式
def generate_csv(account_informations)
  csv_data = CSV.generate(encoding: 'Shift_JIS') do |csv| # 文字コードをShift_JISに指定する必要があります。
    account_informations.each do |account_information|
      sale = Sale.find_by(user_id: account_information.user.id) # 振込金額を先に検索しておきます
      # この時の値の呼び出し方等はアプリに応じて調整してください。

      # GMOあおぞらネット銀行指定形式では、下記項目がこの順番で並んでいる必要があります。
      # 他細かいデータ形式は公式のPDF参照。
      csv << [
        account_information.bank_code,      # 銀行コード　：先頭の0は省略せず4桁で表記
        account_information.branch_code,    # 支店コード　：先頭の0は省略せず3桁で表記
        account_information.deposit_type,   # 預金種目　　：1(普通),2(当座),4(貯蓄)の数値で表記
        account_information.account_number, # 口座番号　　：先頭の0は省略せず7桁で表記
        account_information.bank_account,   # 受取口座名義：半角ｶﾀｶﾅで表記
        sale&.price,                        # 振込金額
        nil,                                # 手数料負担区分（空欄）
        nil,                                # 振り込み依頼人（空欄）
        nil                                 # EDI情報（空欄）
      ]
    end
  end
end
```
:::message
RubyのCSVクラスの特異メソッドである[CSV.generate](https://docs.ruby-lang.org/ja/latest/method/CSV/s/generate.html)を使用しています。
:::
:::message alert
本来であれば、月毎に金額を合計するメソッドなども必要になる可能性がありますが、
今回はCSVエクスポート機能に絞っての説明のため、合計などはせずそのまま値を使用する形とします。
:::



### 2. ルーティングの追加
エクスポートのアクションを実行するためのルーティングを追加します。

```diff_ruby:config/routes.rb
resources :static_pages, only: [:index] do
+ collection do
+   get :export
+ end
end
```

### 3. ビューファイルのCSVエクスポートボタン
先ほど追加したルーティングで生成されたヘルパーを使用します。
```erb:app/views/static_pages/index.html.erb
<%= link_to 'CSVエクスポート', export_static_pages_path(format: :csv), class: 'btn btn-primary w-40' %>
```

## 動作確認
それではCSVエクスポートボタンを押してみます。
（各テーブルのサンプルデータはRailsコンソールで入れておきました。）
![](/images/rails-gmo-csv/image02.gif)
ダウンロードできました！

実際のファイルを確認してみます。
Mac💻だと、Shift_JISの文字コードだと、半角カタカナが文字化けしていたり、
Numbersアプリの仕様で先頭の0が省略されてしまっていますが、問題ありません⭕️
![](/images/rails-gmo-csv/image03.png)
　　
**テキストエディット**で開いた際に、下記画像のように正しい形式で、余計なクォーテーションなどが入っていない状態であれば、GMO銀行でのインポートが可能です！
![](/images/rails-gmo-csv/image04.png)

![](/images/rails-gmo-csv/image05.png)


:::message alert
もしこの形式でもGMO銀行インポート時にエラーが出る際は、
銀行コードや支店コードが実在しないものになっている場合があります。
:::


## まとめ
初めてのRailsのエクスポート機能実装で、銀行でインポート可能な所定の形式に合わせるのが大変でした。
ここまで読んでいただきありがとうございました🙇

　
## 参考・引用記事
#### 引用記事
1）[IT用語辞典 e-Words エクスポート](https://e-words.jp/w/%E3%82%A8%E3%82%AF%E3%82%B9%E3%83%9D%E3%83%BC%E3%83%88.html)
2）[IT用語辞典 e-Words インポート](https://e-words.jp/w/%E3%82%A4%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%88.html)

#### 参考記事
- [エクスポートとは｜「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word1702.html)
- [インポートとは｜「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word1701.html)
- [【Ruby on Rails】CSV出力機能](https://qiita.com/japwork/items/76fb527b1a49d93b7f83)
- [Rails CSVファイルのエクスポート](https://qiita.com/yoshi-sho-0606/items/262f115ee251600527a0)
- [RailsアプリでCSVを出力](https://zenn.dev/d0ne1s/scraps/f0e320ba91ce97)
- [総合振込の全銀フォーマットファイルを作成する方法（Ruby）](https://qiita.com/shjimb/items/44fba0368b00c7c3c404)
- [RubyでShift JISやCP932などのCSVをUTF-8に変換して読み込む](https://qiita.com/daichi87gi/items/9097adfd47d9725097f1)
