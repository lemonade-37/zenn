---
title: "GitHubにAPIキーを載せてしまって大変だった話"
emoji: "🔑"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "GitGuardian", ]
published: true
published_at: 2024-01-31 16:01
---
エンジニア転職を目指して学習しているはると申します。
オンラインスクールでのカリキュラム学習後、個人開発のアプリを作成したのですが、その時にしくじったことについて紹介します🙀

## Google Map APIを導入

初めてのAPI導入。APIキーというのを入れないといけないんだと知り、Google Map APIで取得したAPIキーを設定しました。機密情報だから.envというのを使わないといけないんだな…ということもわかり、しっかりgitignoreに.envも追加しました。

APIを使った一連の機能の実装が完了したため、プルリクエストを作成。個人開発のため、コードレビュー等はせずそのままマージしました。その後、新しいブランチを作成し、次の機能の実装を進めていました。

## GitGuardianからのメール

すると、GitGuardianというところからメールが来ました。内容は英文で、よくわからずとりあえず翻訳してみたところ、
「GitHub アカウント内で 機密情報が公開されています」

えっ🤨
なんで？？

実際にリモートリポジトリの該当のプルリクエストのページを確認すると、.envファイルがプッシュされていて、中身のAPIキーもばっちり見えていました。
あら〜gitignoreに追加したのになんでだろう、どうしようかな〜（ここまで見に来る人あんまりいないだろう）と思い検索していたところ、下記の記事を見つけました。

https://zenn.dev/rabbit/articles/64957d0412ee2e

この記事を読んでやっと事態の重大さに気付く。これ、実務でやったら大問題なことをやってしまったかも？🙀
色々調べても「プルリクエストを完全に削除することはできない」「リモートリポジトリを消すしかない」と出てきました。

とりあえず、APIキーを悪用されないため、リモートリポジトリをpublicからプライベートに変更、APIキーを再生成。

でも履歴上には残ってるのなんかやだな…
リモートリポジトリ上では、卒業試験（README・画面遷移図・ER図）でスクールの講師の方とやりとりした大切な履歴が残っています。
できれば消さないでできる方法はないかと、Twitterで正直に相談したところ、リプライできいなさんより、リポジトリを消さなくてもGitHubに問い合わせをすると機密情報だけを消してもらえると教えていただきました🥺
（それまでリモートリポジトリでミスなく綺麗に使えていたので、プチパニックになっていたので安心しました、本当にありがとうございました🙇）

https://x.com/keynyaan/status/1733104370936664197

## GitHub Supportに連絡

[GitHub Support](https://docs.github.com/ja/enterprise-cloud@latest/support/contacting-github-support) に連絡して、ひとまず現状を伝えました。
下記の旨の返信がありました。

- 一度でもパブリックなリポジトリにプッシュされた機密情報は、履歴を消したとしても、もう情報漏洩していると思ってください。
- すでにマージしたプルリクエストに参照がある場合は、後続のすべてのプルリクエストで機密情報参照できるため、予想よりも多くのプルリクエストから参照を削除する必要がある場合がある。

ここで、本当の意味で事態の重大さに気付きました。
私がやったことは情報漏洩で、前職が医療機関なので情報漏洩の重大さはわかっていましたが、特にプログラミングにおける情報漏洩は、一度でもネットに上がった情報は一生どこで残っているかわからないということを、GitHub Supportからの言葉で実感しました。

そこからはずっと前職でインシデントを起こした後のセーフティカンファレンス時のような気持ちでGitHub Supportとやりとりをしていました。

## 機密情報削除の手順

結論、GitHub Supportに連絡してから3日後に履歴を削除してもらえました。
最初は日本語で送ってしまっていたのと、自分で行う作業があることを知らなかったので、やりとりに時間がかかってしまいました。
下記の手順を行い、翻訳した英文でやりとりすれば、スムーズに対応してもらえました。

### 実際に行った手順について

1. GitHub Supportに連絡して相談を開始した後、最初に機密情報を上げたコミットのコミットIDを聞かれたため伝えました。
GitHub Supportに行ってもらうことと、自分で行わないといけない作業があるため、下記のページを参考に進めました。

https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository

https://thom.hateblo.jp/entry/2021/02/27/145223


2. 下記コード実行。
    ```sh
    % brew install git-filter-repo
    ```

3. 今のアプリと違う場所にリモートリポジトリをクローンし、マニュアルの順序でコードを実行。
    ```sh
    % cd Desktop

    Desktop % git clone リポジトリURL

      Cloning into 'アプリ名'...
      remote: Enumerating objects: 170, done.
      remote: Counting objects: 100% (170/170), done.
      remote: Compressing objects: 100% (115/115), done.
      remote: Total 170 (delta 31), reused 155 (delta 26), pack-reused 0
      Receiving objects: 100% (170/170), 1.21 MiB | 1.21 MiB/s, done.
      Resolving deltas: 100% (31/31), done.


    Desktop % cd アプリフォルダ名

    アプリ名 % git filter-repo --invert-paths --path .env

      Parsed 22 commits
      New history written in 0.06 seconds; now repacking/cleaning...
      Repacking your repo and cleaning out old unneeded objects
      HEAD is now at f7b0723 Merge branch 'develop'
      Enumerating objects: 169, done.
      Counting objects: 100% (169/169), done.
      Delta compression using up to 8 threads
      Compressing objects: 100% (112/112), done.
      Writing objects: 100% (169/169), done.
      Total 169 (delta 30), reused 145 (delta 28), pack-reused 0
      Completely finished after 0.12 seconds.


    アプリ名 % echo ".env" >> .gitignore

    アプリ名 % git add .gitignore

    アプリ名 % git commit -m "Add .env to .gitignore"

      [main dummyid] Add .env to .gitignore
      1 file changed, 1 insertion(+)


    アプリ名 % git remote add origin リポジトリURL

    アプリ名 % git push origin --force --all

      Enumerating objects: 172, done.
      Counting objects: 100% (172/172), done.
      Delta compression using up to 8 threads
      Compressing objects: 100% (113/113), done.
      Writing objects: 100% (172/172), 1.21 MiB | 1.16 MiB/s, done.
      Total 172 (delta 32), reused 167 (delta 30), pack-reused 0
      remote: Resolving deltas: 100% (32/32), done.
      To リポジトリURL
    + dummyid...dummyid develop -> develop (forced update)
    + dummyid...dummyid main -> main (forced update)


    アプリ名 % git push origin --force --tags

      Everything up-to-date
    ```

4. ここまで実行したターミナルログを貼り、GitHub Supportに次に何をしたらいいか質問。

5. 「ここまでやってくれてありがとう、削除できましたよ」のような返事があり、リモートリポジトリを見に行くとちゃんと.env にアクセスできないようになっており、削除できていました！

6. リモートリポジトリと、ローカルリポジトリでだいぶ差分ができてしまったため、ローカルリポジトリを削除し、現在のリモートリポジトリをpullした。（機密情報プッシュに気付いてから行った作業が少なかったため）

## 原因分析・再発予防

では、.envにも追加していたつもりだったのに、なぜこのような事態が発生してしまったのか？を振り返りました🙀

### 原因分析

.envを含む変更をgit add .
→あ、.gitignoreに.env追加してなかった
→追加
→git rm -r --cached **せず** に再度git add .してコミット

**⚠️一度ステージングしてしたファイルは、その後.gitignoreに追加してもステージングされたままであり、git rm -r --cachedコマンドでステージングから削除しなくてはいけなかった**

### 再発予防

- 自分で作成したプルリクは目を通す習慣をつける（実際に働いた時、自分で確認していないものを他人にレビューしてもらうのか？）
- 機密情報の追加など、.envファイルなどを触った後は特に、コミット時は慎重に確認する。
- pushの前に何をステージングしたか、コミットしたか再確認する。

## さいごに

この経験を通して、GitHubの使い方の失敗を身をもって体験し、大変勉強になりました。
今回のこと以外でも、セキュリティ面など、これは本当にやったらまずいということは、勉強しておかなくてはと思いました。
これからも精進します🙇🔥
