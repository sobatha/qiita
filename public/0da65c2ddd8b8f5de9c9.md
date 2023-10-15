---
title: SQL 第2版 ゼロからはじめるデータベース操作
tags:
  - SQL
  - DB
private: false
updated_at: '2023-03-16T01:00:12+09:00'
id: 0da65c2ddd8b8f5de9c9
organization_url_name: null
slide: false
ignorePublish: false
---
## 想定読者
SQL初心者の方、もしくは基礎から見直したいと思っている方
SQL 第2版 ゼロからはじめるデータベース操作を読んで気になった内容のメモ。
また参考としてLeetCodeから関連するSQLの問題へのリンクを貼っています。

# データベースとは
大量の情報を保存しコンピュータから効率よくアクセスできるように加工したデータの集まりのことをデータベースという。また、データベースを管理するコンピュータシステムをデータベースマネジメントサービス、略してDBMSと呼ばれる。
その中でもリレーショナルデータベースと呼ばれる２次元表の形式でデータを管理する DBが現在最も広く利用されている。リレーショナルデータベースにおいてはSQLという専用の言語を用いてデータベースを操作する。具体的には、サーバであるRDBMSにクライアントからSQL文を送信し、RDBMSはその文の内容に従ってデータベースに保存されているデータの読み書きを実行する。なお、リレーショナルデータベースでは、必ず行（レコードとも呼ばれる）単位でデータを読み書きする。

# SQLコマンド
以下主なSQLコマンドを説明する。

## CREATE TABLE
```sql
CREATE TABLE <テーブル名> (
<列名> <データ型> <この列の制約>...
)
```

名前の最初は半角のアルファベットにするのが決まり。
データ型...INTEGER型、CHAR型、VARCHAR型、DATE型　など
制約...NOT NULL、PRIMARY KEY、DEFAULTなど

## SELECT
最も基本となるSQL文。
```sql
SELECT <列名>, ...
FROM <テーブル名>
WHERE <条件式>
```

SELECT句には式や定数を書くことができる。結果から重複行を防ぐにはDISTINCTをつける。WHERE句を指定することで、絞り込み検索を実行できる。
#### LeetCode
[595. Big Countries](https://leetcode.com/problems/big-countries/)

## GROUP BY
```sql
SELECT <列名１>, <列名２>...
FROM <テーブル名>
GROUP BY <列名１>, <列名２>...
```

集約関数とGROUP BY句を使うことで、テーブルを切り分けて集計できる。
集約キーにNULLが含まれる場合には、集計結果にも「不明」行（空行）として現れる。
集約関数とGROUP BY句を使う場合、以下の点に注意する必要がある。
* SELECT句に書けるものが定数、集約関数、GROUP BY句で指定した列名の３つに限定される
* GROUP BY句にはSELECT句でつけた列の別名は使えない
* GROUP BY句は集計結果をソートしない
* WHERE句に集約関数を書くことはできない

最後の点について、集約関数を書くことができるのはSELECT句とHAVING句だけである。WHERE句では行に対する条件指定を、HAVING句ではグループに対する条件指定を書く。WHERE句で条件を指定すると、WHERE句によってソートの前に行を絞り込むことができるため処理速度がHAVINGを利用するよりも高速化できる。また、インデックスをしてしてWHERE句を利用した場合には処理が大幅に高速化される。

#### LeetCode
[1050. Actors and Directors Who Cooperated At Least Three Times](https://leetcode.com/problems/actors-and-directors-who-cooperated-at-least-three-times/)
[586. Customer Placing the Largest Number of Orders](https://leetcode.com/problems/customer-placing-the-largest-number-of-orders/)

## サブクエリ
サブクエリを利用することで、複数のクエリを組み合わせて１つのSQLを発行することができる。
#### LeetCode
[183. Customers Who Never Order](https://leetcode.com/problems/customers-who-never-order/description/)

## CASE式
```sql
CASE WHEN <評価式> THEN <式>
     WHEN <評価式> THEN <式>...
     ELSE <式>
END
```
評価式とは、戻り値が心理値になる式のことで、=,!=,LIKE,BETWEENなどの述語を使って作る式のこと。
ELSE句は省略可能であるが、省略しないほうがわかりやすい。END句は省略不可。
CASE式は式であるので、式をかけるところであればどこにでも書ける。
#### LeetCode
[627. Swap Salary](https://leetcode.com/problems/swap-salary)

## JOIN
```sql
SELECT <FROMで指定した別名orテーブル名>.<列名１>, <FROMで指定した別名orテーブル名>.<列名２>...
FROM <テーブル名1> AS <別名> 
    JOIN <テーブル名2> AS <別名> 
    ON <テーブル名1or別名>.<結合キー> = <テーブル名1or別名>.<結合キー>
```
JOINとは別のテーブルから列を持ってきて列を増やす集合演算である。基本は内部結合（INNER JOIN）と外部結合（LEFT JOIN, RIGHT JOIN）の２つ。

#### LeetCode
[607. Sales Person](https://leetcode.com/problems/sales-person/)
[181. Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers/)
[1084. Sales Analysis III](https://leetcode.com/problems/sales-analysis-iii/)

# トランザクションとは
トランザクションとは、データベースに対する１つ以上の更新をまとめて呼ぶときの名前である。
 
```sql
BEGIN TRANSACTION; --MySQLではSTART TRANSACTION
 --DML文①;
 --DML文②;...

COMMIT; --もしくはROLLBACKにてロールバック実行
```

ただし、一般的なDBMSにおいてはトランザクションの開始コマンドは不要であり、DBに接続した時点で暗黙にトランザクションが開始される。これには以下の２パターンがある。
1. 「１つのSQL文で１つのトランザクション」というルールが適用される（自動コミットモード）
2. ユーザがCOMMITもしくはROLLBACKを実行するまでが１つのトランザクションと見做される。

## ACID特性
- **原子性 Atomicity**
    - トランザクションが終わったとき、含まれていた全ての処理が実行されるか、全て実行されない状態で終わるかを保証する性質のこと。トランザクションはひとかたまりとして扱われ、一部だけが実行され、残りは未実行のままという結果にはならないことが保証される。
- **一貫性 Consistency**
    - トランザクションに含まれる処理は、データベースにあらかじめ設定された制約を満たすという性質のこと。制約を満たさない違法なSQLはロールバックされる。整合性と呼ばれることもある。
- **独立性 Isolation**
    - トランザクション同士が互いに干渉を受けないことを保証する。この性質により、トランザクション同士が入れ子になるようなことはない。また、トランザクションによる変更はトランザクション終了時まで他のトランザクションから隠蔽される。
- **永続性 Durability**
    - トランザクションが終了したら、その時点でのデータの状態が保存されることを保証する性質。たとえシステム障害によりデータが失われたとしても、データベースはトランザクションの実行記録をログなどに保存しておき、このログを使って障害前の状態に復旧する。

# NULLの取り扱い
* NULLを含む計算を行うと、結果はNULLになる。
* IS NULLもしくはIS NOT NULLを用いてNULLかどうかの比較を行う。
* COUNT関数は引数によって動作が異なる。COUNT(*)ではNULLを含んだ行数を、COUNT(<列名>)ではNULLを除外した行数を数える。COUNT以外の集約関数はNULLを除外する。
* NULLを値へ変換したいときには、COALESCE（コアレス、もしくはコォアリース）関数を使う。COALESCE関数は左から順に引数を見て、最初にNULLではない値を返す。
* NOT INの引数にNULLが含まれる場合、常に結果は空となり１行も選択されない。


## 参考
[SQL 第2版 ゼロからはじめるデータベース操作](https://www.amazon.co.jp/SQL-%E7%AC%AC2%E7%89%88-%E3%82%BC%E3%83%AD%E3%81%8B%E3%82%89%E3%81%AF%E3%81%98%E3%82%81%E3%82%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E6%93%8D%E4%BD%9C-%E3%83%9F%E3%83%83%E3%82%AF-ebook/dp/B01HD5VWWO/ref=sr_1_6?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=JM6BLDDV8UIF&keywords=SQL&qid=1674912080&sprefix=sql%2Caps%2C229&sr=8-6)

IT各社の新入社員研修資料より
[今年もミクシィの22新卒技術研修の資料と動画を公開します！](https://mixi-developers.mixi.co.jp/22-technical-training-5fc362a9dc41#b523)
[はてなリモートインターンシップ2022 RDBMSブートキャンプ 講義資料](https://speakerdeck.com/hatena/remote-internship-2022-rdbms)

よりよいSQLを書きたい！という人向け
[あなたの遅延はどこから？ SQLから！ 〜患部に止まってすぐ効くSQLレビューチェックリスト 年初め特大サービス号〜](https://tech.andpad.co.jp/entry/2023/01/12/100000)
[66分かかる同期処理を10分以内に短縮せよ！～商品情報同期システムでの、処理速度と運用の改善～](https://tech-blog.monotaro.com/entry/2022/08/23/090000)

大学の講義資料　すごくやってみたいとは思っている。思っている。。。でも教科書がまず1万円するんだよ。日本のアマゾンだと。だれかー
[CMU 15-445/645FALL 2022 Database Systems](https://15445.courses.cs.cmu.edu/fall2022/)
