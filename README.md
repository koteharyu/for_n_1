# README

## Introduction
N+1対策の知識は高負荷railsアプリケーション開発に必須のスキルです。
市場のrails案件は高負荷(急成長サービス、大規模サービス、高トラフィックサービス)も多く、また、今は小規模でも急成長する可能性もあります。

高トラフィックサービスの運用には、スケーラビリティ性が必須です。
N+1とはデータと計算量が比例して増えてしまう状態。つまり、スケーラビリティ性が低い状態なので高トラフィックを捌けない、または将来捌けなくなるコードという状態です。

スケーラビリティ性はインフラもアプリケーションアーキテクチャもコーディングテクニックも必要です。
アプリケーションのN+1対策の知識はコーディングテクニックの話に該当します。

railアプリケーションで高負荷システム開発に現在携わっている、携わりたい、これから携わる人の為に
ページ表示に5秒以上かかっているページに以下の3つのメソッドを正しく使い、0.4秒までパフォーマンスチューニングするリポジトリを作成しました。

* joins
* preload
* eager_load 

## Middleware versions

| middleware | version |
|---|---|
| Ruby | 2.3.0p0 |
| Rails | 6.1.0 |
| node.js | v12.16.2 |
| yarn| 1.22.4 |

## Degign

<img width="1434" alt="スクリーンショット 2020-05-01 0 25 50" src="https://user-images.githubusercontent.com/18366969/80728952-82fde080-8b42-11ea-907b-1c71e124866f.png">

## ER diagram
<img width="1434" alt="ER diagram" src="https://user-images.githubusercontent.com/18366969/80870837-ea9f6180-8ce3-11ea-92a0-967407d97af8.png">

## branch

| branch | 役割 | ページ描画速度 |
|---|---|---|
| master | チューニング前 | 5000ms以上 |
| master_tuning | チューニング後 | 400ms以下 |
| use_bullet | gem 'bullet'のサンプル | 400ms以下 |

## Setup

```
$ git clone https://github.com/soartec-lab/rails_tuning_sample_app.git
$ cd rails_tuning_sample_app
$ bundle install
$ yarn
$ rails db:create
$ rails db:migrate
$ rails db:seed
$ rails s
```

# N+1

## 通例

User has many articles

名前が`rails`であるarticleの著者を算出したい

## methods

### joins

引数に関連付のシンボルを指定することでレシーバーのモデルと、関連モデルを統合することができる

```rb
User.joins(:articles).where('articles.name = ?', 'rails')
```

使用する場面...`関連テーブルでの絞り込み`

### left_outer_joins

関連テーブルにレコードが存在しない場合にも統合元のレコードは全て取得する

```rb
User.left_outer_joins(:articles).where('articles.name = ?', 'rails')
```

使用する場面...`関連テーブルでの絞り込み`

### eager_load

関連付を一括読み込みするメソッド。

引数に関連付けのシンボルを指定することで、レシーバーのモデルと関連モデルを統合することができる

```rb
User.eager_load(:articles).where('articles.name = ?', 'rails')
```

1. 初回に、SQLの`LEFT OUTER JOIN句`を発行し関連付をキャッシュする
2. ２回目以降はDBではなく、メモリのキャッシュを使って関連データを取得する
3. よってSQLが発行されずN＋1が起こらない

使用する場面...`ループ内で関連テーブルの値を使用する場合`

### preload

関連付を一括読み込みするメソッド

SQLをモデルごとに発行し、`eager_load`同様、関連付をキャッシュする

対象テーブルのデータサイズが大きくJOINのコストが大きい場合に有効

Userテーブル、Articleテーブルそれぞれに１回ずつSQLが発行される

`SELECT句をモデルごとに１回ずつ発行する`

使用する場面...`ループ内で関連テーブルの値を使用する場合`

### includes

SQLの発行時の条件に合わせて`eager_load`, `preload`どちらかの挙動をする

ちなみに、以下の条件に合致する場合には`eager_load`の挙動を、合致しない場合は`preload`の挙動をとる

1. 引数に指定した関連テーブルに対し`joins`メソッドを使用している場合
2. 引数に指定した関連テーブルに対し`where`メソッドを使用している場合
3. 引数に指定した関連テーブルに対して`references`メソッドを使用している場合

<br>

## eager_load, preloadのメリット・デメリット

どちらともメモリにキャッシュを残すため、再利用しないデータはキャッシュしないように注意。

関連テーブルでの絞り込みのみ行う場合は、`joins` or `left_outer_joins`を火なら使うこと

### eager_load

結合するテーブルサイズに注意する必要がある

#### メリット

`preload`とは異なりJOIN句を使うため、関連テーブルでの絞り込みが可能となりSQL発行が１回で済む

#### デメリット

RDBはJOIN句の処理に時間がかかるもの。そのため対象テーブル全体のカラム数や格納されているデータ量など1レコードあたりのデータサイズや単純なレコード数により処理が遅くなる場合がある。

また、RDB側で大きなデータ量を使おうとしてメモリ不足になりディスクを使用することでスロークエリになってしまう可能性もある

<br>

### preload

IN句に指定される要素の数、つまり関連レコード数に注意する必要がある

#### メリット

`preload`は関連モデルごとにSQLを発行するため、`eager_load`で発生するJOIN句特有の問題を回避することができる

#### デメリット

が、SQLを関連モデルごとに発行することで、`eager_laod`よりもSQL発行数が多くなってしまう。

また、RDBによっては１つのSQL内のIN句に指定できる数や、一度に処理できるSQLのバイト数の設定に上限があるため、IN句が長くなることで予期しないRDB側のエラーが発生する可能性がある

