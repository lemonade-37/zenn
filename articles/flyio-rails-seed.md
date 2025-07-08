---
title: "Fly.ioでrails db:seedを実行する方法"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "Rails", "Flyio"]
published: true
published_at: 2023-12-16 12:00
---
## 自己紹介
はじめまして、はると申します。現在はスクールに通いながら学習をしています🐥


## 概要
Fly.ioでRailsアプリをデプロイした際に、`rails db:seed`を実行する方法。

### 環境
- Ruby: 3.2.2
- Rails: 7.0.8

:::message alert
うまく行った方法の1つを紹介していますが、より良い方法を知っている方がいたら教えて頂けると嬉しいです🙇
:::


## 結論
RAMサイズを`256MB`から`512MB`に上げた後、以下のコマンドでうまくいきました。
```shell
hogehoge@hogehogenoMBP アプリ名 % flyctl console

irb(main):001:0> Rails.application.load_seed
=> true
```

## 詳細
Fly.ioでのseeds.rbの反映方法について調べて見つけたコマンドを実行しましたが、下記のようなエラーが出て実行できませんでした。

```shell
hogehoge@hogehogenoMBP アプリ名 % fly ssh console
Error: app アプリ名 has no started VMs.
It may be unhealthy or not have been deployed yet.
Try the following command to verify:
```

コンソールに入って直接`rails db:seed`を実行も試してみましたが、下記のようなエラーになりました。
```shell
hogehoge@hogehogenoMBP アプリ名 % flyctl ssh console -C "app/bin/rails db:seed"
Error: app アプリ名 has no started VMs.
Try the following command to verify:

fly status

hogehoge@hogehogenoMBP アプリ名 % fly console
irb(main):001:0> /rails/bin/rails db:seed
/usr/local/lib/ruby/3.2.0/irb/workspace.rb:119:in `eval': (irb):1: unknown regexp option - b (SyntaxError)
/rails/bin/rails db:seed
      ^~~~
(irb):1: syntax error, unexpected label, expecting `do' or '{' or '('
                 ^~~

irb(main):002:0> rails db:seed
(irb):2:in `<main>': undefined local variable or method `seed' for main:Object (NameError)
Did you mean?  send
```

ここで、RAMサイズを256MBから512MBに増やさないとRailsコンソールが使えないという質問ページ[1)](#引用記事)を見つけたので、公式ドキュメント[2)](#引用記事)に沿ってRAMを512MBにしました。

```shell
hogehoge@hogehogenoMBP アプリ名 % fly scale show
VM Resources for app: アプリ名

Groups
NAME    COUNT   KIND    CPUS    MEMORY  REGIONS
app     2       shared  1       1024 MB nrt(2)


hogehoge@hogehogenoMBP アプリ名 % fly scale vm shared-cpu-2x
Updating machine 12345678900000
No health checks found
Machine 12345678900000 updated successfully!
Updating machine 00000987654321
No health checks found
Machine 00000987654321 updated successfully!
Scaled VM Type to 'shared-cpu-2x'
      CPU Cores: 2
         Memory: 512 MB


hogehoge@hogehogenoMBP アプリ名 % fly scale show
VM Resources for app: アプリ名

Groups
NAME    COUNT   KIND    CPUS    MEMORY  REGIONS
app     2       shared  2       512 MB  nrt(2)
```

その後、再度seedのコマンドを実行しましたが下記のエラーに変わりました。

```shell
hogehoge@hogehogenoMBP アプリ名 % flyctl ssh console -C "/app/bin/rails db:seed"
Connecting to fdaa:0:0000:000:000:0000:0000:0... complete
fork/exec /app/bin/rails: no such file or directory
Error: ssh shell: wait: remote command exited without exit status or exit signal
```

ここで、別の質問ページ[3)](#引用記事)で見つけたコマンドを実行してみたところ、seedデータが反映されました。
```shell
hogehoge@hogehogenoMBP アプリ名 % flyctl console
Created an ephemeral machine 12345678900000 to run the console.
Connecting to fdaa:0:0000:000:000:0000:0000:0... complete
Loading production environment (Rails 7.0.8)
irb(main):001:0> Rails.application.load_seed
=> true
```

## VMとは
- 仮想機械（かそうきかい、仮想マシン、バーチャルマシン、英語: virtual machine、VM）とは、アプリの使用を最適化する方法であり、コンピュータの動作を再現するソフトウェアである。すなわち、エミュレートされた仮想のコンピュータそのものも仮想機械という。[4)](#引用記事)

Fly.ioでは、デプロイコマンド(`fly launch`と`fly deploy`)を実行すると、Dockerfileの作成とイメージビルドが行われ、アプリとDBのコンテナが作成されるため、
その仮想マシンのことを指しているという認識です。


## RAMとは
- Random-access memory（ランダムアクセスメモリ、RAM、ラム）とは、コンピュータで使用するメモリの一分類である。
- 「ランダムアクセス・メモリ」とは、任意のアドレスの記憶素子に対して随時、アクセスパターンに依存した待ち時間などを要することなく、読み出しや書き込みといった操作ができるメモリを指す語である。
- 本来は、格納されたデータに任意の順序でアクセスできる（ランダムアクセス）メモリといった意味で、かなりの粗粒度で「端から順番に」からしかデータを読み書きできない「シーケンシャルアクセスメモリ」(SAM)と対比した意味を持つ語であった。しかし本来の意味からズレて、電源を落としても記録が消えないROM（これも本来の読み出し専用メモリからは意味がズレてきている。）に対して、電源が落ちれば記憶内容が消えてしまう短期メモリの意で使われていることが専らである。[5)](#引用記事)

Fly.ioでは、`fly scale vm`コマンドで、事前に設定されたCPU/RAMの組み合わせが適用されます。
一度にたくさんの処理をするには、RAMやCPUのサイズが大きい必要があります。
Railsコンソールを実行するには、RailsサーバーとDBの両方を同時に実行するため、RAMが足りなくてエラーが起こっていたのではないかという認識です。


## Rails.application.load_seedとは

初めて見たコマンドだったので調べてみました。
今までターミナルで`rails db:seed`を実行するだけでseedデータが反映されていたのは、Rails内で色々な処理が行われていたということがわかりました。
下記の記事が詳しく説明されていたので、引用させていただきます🙇

https://qiita.com/takoba/items/c0907a4c0ec8b51eb2a4

今回の場合では、ターミナル上で`rails db:seed`を実行したときに行われる処理の中の一つを、
railsコンソールに入り、直接コマンドで実行した形でした。


## まとめ
これ以外のより良い方法や、SSH接続できなかった理由まではわかりませんでしたが、
同じエラーで困っている方のために方法の一つとして共有でした🙇
コンピュータの基礎やデプロイ周辺の知識が浅いのでこれからも学習していきたいと思います。

　
## 引用記事
1）[Trouble access rails console](https://community.fly.io/t/trouble-access-rails-console/4511)
2）[Scale Machine CPU and RAM（公式ドキュメント）](https://fly.io/docs/apps/scale-machine/)
3）[Launch and deploy success but database seems empty, may need help with seeding](https://community.fly.io/t/launch-and-deploy-success-but-database-seems-empty-may-need-help-with-seeding/8314/3#:~:text=Rails%20%E3%82%B3%E3%83%B3%E3%82%BD%E3%83%BC%E3%83%AB%E3%81%A7%20Rails.application.load_seed%20%E3%82%92%E5%AE%9F%E8%A1%8C%E3%81%97%E3%81%9F%E3%81%A8%E3%81%93%E3%82%8D%E3%80%81%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%81%8C%E3%82%B7%E3%83%BC%E3%83%89%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%E3%82%88%E3%81%86%E3%81%AB%E8%A6%8B%E3%81%88%E3%81%BE%E3%81%97%E3%81%9F%E3%81%8C%E3%80%81%E4%B8%8D%E5%AE%8C%E5%85%A8%3A)
4）[Wikipedia 仮想機械](https://ja.wikipedia.org/wiki/%E4%BB%AE%E6%83%B3%E6%A9%9F%E6%A2%B0)
5）[Wikipedia Random Access Memory](https://ja.wikipedia.org/wiki/Random_Access_Memory)
6）[`rake db:seed`の処理の流れを追った](https://qiita.com/takoba/items/c0907a4c0ec8b51eb2a4)
