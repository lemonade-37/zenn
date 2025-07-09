---
title: "タイポ発見ツールtyposをGitHub Actionsで導入してみた"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typos", "GitHubActions"]
published: true
published_at: 2024-04-30 12:00
---
## 自己紹介
はじめまして、はると申します。
駆け出しエンジニアとして働き始めて約2ヶ月が経過しました🐣


## 概要
CI/CDの知識がない駆け出しエンジニアが、タイポ発見ツールの`typos`をGitHub Actionsで設定してみました。

### 環境
- PC内にhomebrewはすでにインストール済み
- ローカル環境・Dockerなし
- Ruby3.2.2、Rails7.0.8


## typosのインストール

下記コマンドを実行し、ローカルPC内に`typos`をインストールします。
```c:ターミナル
brew install typos-cli
```

これで、アプリ内のディレクトリで
```
typos
```
と実行すると、タイポを発見してくれるようになりました。

:::details 実行例😱
沢山タイポが見つかりました…

```c:ターミナル
MBP hokkaido_travel % typos
error: `seach` should be `search`
  --> ./app/javascript/forgot_spotname.js:3:9
  |
3 |   const seachButton = document.getElementById('search-btn')
  |         ^^^^^
  |
error: `seach` should be `search`
  --> ./app/javascript/forgot_spotname.js:5:3
  |
5 |   seachButton.addEventListener('click', function(event) {
  |   ^^^^^
```
:::

## GitHub Actionsでプルリクのタイポチェックをする

では、これをGitHub Actionsに設定し、プルリクエストを作成するたびにタイポをチェックしてくれるようにします。

### GitHub Actionsとは
リポジトリ内のイベント（プッシュなどの処理）によってトリガーされたときに実行される処理をあらかじめ設定しておき、自動で実行してくれるようにします。
今回のタイポチェックツールの他にも、自動でRSpecを実行するなど、さまざまな用途で使用されます。

### 実装方法

1. WebブラウザでGitHubのリポジトリページにアクセスします。
`Actions`タブを開き、`Set up a workflow yourself →`をクリックします。
![](/images/typos-github-actions/image01.png)
　　
1. ファイル名を`typos.yml`、内容は下記のように入力し、コミットします。ファイルが作成できたらOKです。
    ```yml:typos.yml
    name: GitHub Action
    on: [pull_request]

    jobs:
      run:
        name: Spell Check with Typos
        runs-on: ubuntu-latest
        steps:
          - name: Checkout Actions Repository
            uses: actions/checkout@v3

          - name: Run Typos
            uses: crate-ci/typos@master
    ```

    ![](/images/typos-github-actions/image02.png)
    　　
1. ローカル環境で、`git pull origin main`しておきます。
※ 1 と 2 の手順は、ローカル上でファイル作成して push でもOKです。
　　
1. ローカル環境、アプリのディレクトリ内で下記コマンドを実行します。
    ```c:ターミナル
    アプリ名 % touch .typos.toml
    ```
    　　
1. 作成したファイル内を編集します。
    ```toml:.typos.toml
    # スペルチェックを無効にするフォルダやファイルの設定
    [files]
    extend-exclude = [
      "*.svg",
      "node_modules",
      "vendor/**/*",
      "**/*.min.js",
      "logs/**/*",
      "*.png",
      "*.jpg",
      "*.gif",
      "*.ico",
      "*.exe",
      "build/**/*",
      "dist/**/*"
    ]
    ignore-hidden = true # 隠しファイルを無視
    ignore-vcs = true    # バージョン管理システムのファイルを無視

    # 一般設定
    [default]
    check-filename = true  # ファイル名のスペルチェックを有効
    check-file = true      # ファイル内容のスペルチェックを有効
    locale = "en-us"       # 英語のロケールをアメリカ英語に指定

    # もし他にも設定したい場合は、下記のように設定できます。
    # プロジェクト内で使用される独自の技術用語や、英国英語のスペルなど、default.extend-wordsで設定すると、タイポとして扱われなくなります。
    [default.extend-words]
    colour = "color"              # 「colour」という単語を検知して「color」に直すようにエラーを出すようにする設定
    initialise = "initialize"     # 「initialise」という単語を検知して「initialize」に直すようにエラーを出すようにする設定
    analogue = "analog"           # 「analogue」という単語を検知して「analog」に直すようにエラーを出すようにする設定
    rememberable = "rememberable" # 左右を同じにすることで、「rememberable」という単語はスペルミスとして扱われないように除外できる

    # など、自由にカスタマイズできます。
    ```
    他にも細かく設定したい方は公式ドキュメントの[typos/docs/reference.md](https://github.com/crate-ci/typos/blob/master/docs/reference.md)を参照ください。
    　　
1. 作成できたら、`add`・`commit`・`push`します。
    　　
1. 試しに、わざとタイポしてプルリクエストを作成し、引っ掛かるかどうか確認してみます。
    プルリクエスト内で、GitHub Actionsが実行されタイポが検知されたことが確認できます。
    ![](/images/typos-github-actions/image03.png)
    　　
    `Details`を開くと、どのファイルの何行目でタイポしているかが確認できます。
    これで、プルリクを作成するたびに、タイポがないか確認できるようになりました🎉

    ![](/images/typos-github-actions/image04.png)


## まとめ
ふんわり自動でテスト実行してくれるもの？くらいの理解だったGitHub Actionsを実際に触ってみて、少し理解が深まりました。
このような便利なツールを、タイポの対策などさまざまな用途で活用していけるように、これからも勉強していきます💻🔰

　
## 参考記事
- [crate-ci/typos: Source code spell checker](https://github.com/crate-ci/typos)
- [固有名詞の除外登録の手間いらず！typosを使ってお手軽スペルチェック ~GitHub Actions編~](https://zenn.dev/tksfjt1024/articles/3901a1e4756173?redirected=1)
- [GitHub Actions を理解する - GitHub Docs](https://docs.github.com/ja/actions/learn-github-actions/understanding-github-actions#workflows)
- [GitHub Actionsって何？触ってみて理解しよう！入門・逆引きリファレンス](https://qiita.com/yu-ichiro/items/b50ceb0008edc3c0312e)
