---
title: "RuboCop を GitHub Actions に導入した"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "RuboCop", "Linter", "GitHubActions"]
published: true
published_at: 2024-09-10 12:00
---
## 自己紹介
はじめまして、はると申します。
エンジニアとして働き始めて約半年が経過しました🐣


## 概要
新規プロジェクト開始時に、Rails アプリ内に RuboCop を導入し、GitHub Actions に設定する手順をまとめました。

### 環境
- Ruby 3.3.4
- Rails 7.1.3.4

の環境で動作確認済み

## なぜ RuboCop が必要なのか、自分なりの考え
### なぜ RuboCop というツールを使うのか
プロジェクト内のコーディング規約を、RuboCop のカスタマイズ設定としておけば、レビューでいちいち指摘するまでもないようなことも各自で修正できますし、後からプロジェクトに入った方も申し送りせずとも同じルールで記載ができます。

また、厳しくコードレビューを行っていても、人間なので見落とす可能性はゼロにはできません。
1つの文法ミスがアプリ動作時はエラーになっておらず、インフラの作業時にエラーとして現れ、なかなか原因に気づけなかった経験が自分もあります…。

### なぜコーディング規約や、綺麗なコードである必要があるのか
自分の考えですが、綺麗なコードは、プロジェクト内で書き方が統一されており、コードを見れば何をしているのかわかるコードだと考えました。
一目で見てわかりやすいコードであれば、コードを読むコストが減るので、お客様やユーザーの追加機能などの要望に素早く応えたりと、素早く価値を提供できると考えます。
（RuboCop でチェックできる範囲にとどまらない話になるので詳細は割愛します）


## RuboCop の導入
1. Gemfile に gem rubocop を追加し、 `bundle install` コマンドを実行します。
    ```diff_ruby:Gemfile
    group :development, :test do
      # See https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem
      gem "debug", platforms: %i[ mri windows ]
    + gem 'rubocop'
    + gem 'rubocop-rails'
    end
    ```

    それぞれの Gem の機能は以下の通りです。
    - [rubocop](https://github.com/rubocop/rubocop)：Ruby スタイルガイドに基づいて Ruby 全体のコーディングをチェックします。
    - [rubocop-rails](https://github.com/rubocop/rubocop-rails)：Rails のベストプラクティスとコーディング規則の実施に重点を置いた RuboCop の拡張機能です。
    　　
1. Gem が導入できたら、以下コマンドを実行します。
    ```terminal
    (local)$ rubocop --auto-gen-config
    ```
    （Docker 環境の場合は `docker compose exec <アプリコンテナ> 〜` をつけて実行して下さい）
    　　
1. コマンドを実行すると、2つのファイルが生成されます。
    - .rubocop.yml：
    RuboCop のチェック項目のカスタマイズのための設定ファイルです。特定の項目の除外や、項目ごとに厳しさの度合いを設定することができます。
    - .rubocop_todo.yml：
    `rubocop --auto-gen-config` コマンドを実行した時点での RuboCop の違反がこのファイルに記載され、RuboCop のチェックから除外されます。
    RuboCop 違反を一度に修正するのは大変なため、.rubocop_todo.yml ファイルの項目を1つずつ削除して修正していきます。
    コードが変わった場合は、再度 `rubocop --auto-gen-config` を実行し最新時点での .rubocop_todo.yml ファイルを再生成する必要がある場合があります。
    違反がなければ、空になります。
    　　
1. 上記ファイルを生成せずにチェックを実行したい場合は、以下コマンドで実行できます。
    ```terminal
    (local)$ rubocop
    ```
    　　
1. .rubocop_todo.yml に違反が記載された場合は修正や、.rubocop.yml ファイルをカスタマイズして除外設定を追加するなど、プロジェクト内の方針に沿って設定を行って下さい。


## GitHub Actions への追加方法
1. プロジェクト内に .githubフォルダを作成し、その中に workflows フォルダを作成し rubocop.yml ファイルを作成します。
　　
1. GitHub Action 用の記載をします。以下は一例です。
    ```yml:.github/workflows/rubocop.yml
    name: Check RuboCop
    on:
      push:
      pull_request:

    jobs:
      rubocop:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v3

          - name: Set up Ruby
            uses: ruby/setup-ruby@v1
            with:
              bundler-cache: true

          - name: Run RuboCop
            run: bundle exec rubocop
    ```
    　　
1. 上記設定であれば、プルリクエスト作成時と、プッシュ時に GitHub Action で RuboCop が走るようになります。
    違反があると画像のようになり、`Details` からどの部分で引っ掛かっているのか詳細を確認することができます。
    GitHub 上のリポジトリ設定で、全てをパスしないとマージできない設定にすること等もできます。

    ![](/images/rubocop-github-actions/image.png)



## まとめ
新規リポジトリを作成するたびに導入作業を行うので、備忘録を兼ねてまとめました💎
ここまで読んでいただきありがとうございました！

　
## 参考・引用記事
#### 引用記事
1）[rubocop/rubocop](https://github.com/rubocop/rubocop)
2）[rubocop/rubocop-rails](https://github.com/rubocop/rubocop-rails)

#### 参考記事
- [【Rails】rubocop導入手順の備忘録](https://qiita.com/kumaryoya/items/e2e12bb503fe56404329)
- [RailsにGitHub Actionsの導入（Rspec, Rubocop）](https://zenn.dev/ryouzi/articles/cd6857c08e60e7)
- [GitHubActionsによるRSpec・RuboCopのCI設定](https://qiita.com/k-sukesakuma/items/f6fc0968113a77e80d0e)
- [rubocopによる静的コード解析でRubyのコード品質を保つ](https://hiroki.jp/rubocop)
