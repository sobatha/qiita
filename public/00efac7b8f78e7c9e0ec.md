---
title: DjangoのManagerとQuerySet、どっちがいいの問題
tags:
  - Python
  - Django
private: false
updated_at: '2022-11-18T18:17:35+09:00'
id: 00efac7b8f78e7c9e0ec
organization_url_name: null
slide: false
ignorePublish: false
---

# 今回の問題
ManagerメソッドとQuerySetからas_managerでManagerとして利用すること、どっちが適切なのか？
というのが不安だったので、自分用にメモ。
# 問題の背景
あんまりManagerとQuerySetの違いから理解できていない。
# Modelとは
そもそもModelとは何かまで遡る。
Djangoの公式ドキュメントによると
> モデルは、データに関する唯一かつ決定的な情報源です。あなたが保持するデータが必要とするフィールドとその動作を定義します。一般的に、各モデルは単一のデータベースのテーブルに対応付けられます。
>基本:
> * モデルは各々 Python のクラスであり django.db.models.Model のサブクラスです。
> * モデルの属性はそれぞれがデータベースのフィールドを表します。
> * これら全てを用いて、Django はデータベースにアクセスする自動生成された API を提供します。 

つまり、Modelとはデータベースのテーブルを定義するもの。
役割としてはデータとPythonオブジェクトを定義するFieldsを運ぶ。
これによってデータがFieldsのオブジェクトクラスに対応されたモデルインスタンスをQueryから発行できる。
ちなみにFieldsはデータベースのカラムを意味していて、制約やデータの種類を定義する。
> オブジェクトに独自の "行レベルの" 機能を追加するには、カスタムのメソッドを定義してください。
Managerにカスタムの@propartyなどを利用して行レベルの機能追加ができる。
例えば、
```python: model.py
class Item(Models.Model):
    name = models.CharField(min_length=3, max_length=127)
    price = models.DecimalField(max_digits=17, decimal_places=2)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE
    )

    @proparty
    def price_tax_included(self):
        return self.price * (1+TAXRATE)
```
関数を定義するよりも、propartyにしておく方がアクセス、利用がやりやすくなる。
その分計算の回数も増えるので、気になる時はなんらかcasheしておくとよい。

# Managerとは
Django公式によると、
> マネージャ (Manager) とは、Django のモデルに対するデータベースクエリの操作を提供するインターフェイスです。Django アプリケーション内の1つのモデルに対して、Manager は最低でも1つは存在します。
> デフォルトでは、Djangoはobjectsという名前のManagerを各Djangoモデルクラスに対して追加します。
> Manager はモデルのインスタンスでなく、モデルのクラスを経由してのみアクセスでき、それは "テーブル水準" の処理と "レコード水準" の処理とで責任を明確に分離するためです。

ManagerはModelへの操作を行う入り口。
カスタム可能。後で述べる。
> Managerメソッドは "テーブル単位の"操作をするように意図されており、モデルメソッドは特定のモデルインスタンス上で動作します。

モデルメソッドのリストは[ここ](https://docs.djangoproject.com/ja/4.1/ref/models/instances/#model-instance-methods)

# QuerySetとは
> データベースからオブジェクトを取得するには、モデルクラスのManagerからQuerySetを作ります。
> QuerySetはデータベース上のオブジェクトの集合を表しています。多数のフィルターを持つことができます。フィルターは与えられたパラメータに基づいてクエリの検索結果を絞り込みます。SQL文においては、QuerySetはSELECT句、フィルターはWHEREやLIMITのような絞り込みに用いる句に対応しています。
> モデルの Managerを用いることで QuerySetを取得します。

Managerから呼び出されるオブジェクトの集合。
SQLクエリの構築方法やレスポンスをカスタマイズする必要がある場合は、クエリセットが適切な場所。

QuerySetのメソッドリストは[ここ](https://docs.djangoproject.com/ja/4.1/ref/models/querysets/#methods-that-do-not-return-querysets)


# Managerに新しいメソッドを追加する
> Managerのカスタマイズメソッドは、QuerySetでもそれ以外でも、どんなオブジェクトを返しても構いません。[^1]


例としては
```python: model.py
class ItemManager(models.Manager):
    def on_sale(self) -> QuerySet:
        queryset = self.get_queryset()
        return queryset.filter(released_at__lte=datetime.now())

    def female(self):
        queryset = self.get_queryset()
        return queryset.filter(category__name='female')
        

class Item(Models.Model):
    name = models.CharField(min_length=3, max_length=127)
    price = models.DecimalField(max_digits=17, decimal_places=2)
    released_at = models.DateTimeField(null=True, blank=True)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE
    )

    objects = ItemManager()
```
以上で発売中の商品を確認したいときに
Item.objects.on_sale()でQuerySetが返ってくる。
ただし、Item.objects.on_sale().female()はできない（QuerySetに対してManagerメソッドは使えない）。

## Pros
Managerそのものを書き換えられる。
## Cons
Managerメソッドのため、QuerySetに対して利用することはできない。
重ねて使いたい場合には、classmethodでManagerを返すメソッドの後にしか使えない。
# QuerySet as_managerでクエリを返す方法
```python: model.py
class ItemQuerySet(models.QuerySet):
    def on_sale(self) -> QuerySet:
        queryset = self.get_queryset()
        return queryset.filter(released_at__lte=datetime.now())
    
    def female(self):
        queryset = self.get_queryset()
        return queryset.filter(category__name='female')

class Item(Models.Model):
    name = models.CharField(min_length=3, max_length=127)
    price = models.DecimalField(max_digits=17, decimal_places=2)
    released_at = models.DateTimeField(null=True, blank=True)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE
    )

    objects = ItemQuerySet.as_manager()
```
以上でManagerカスタムの時と全く同じで
Item.objects.on_sale()でQuerySetが返ってくる。
## Pros
QuerySetメソッドなので、重ねて利用できる。
つまり、Item.objects.on_sale().female()が可能。
## Cons
すべてのManagerメソッドがコピーされるわけではないぽい。
> Not every QuerySet method makes sense at the Manager level; for instance we intentionally prevent the QuerySet.delete() method from being copied onto the Manager class.
> Methods are copied according to the following rules:
> * Public methods are copied by default.
> * Private methods (starting with an underscore) are not copied by default.
> * Methods with a queryset_only attribute set to False are always copied.
> * Methods with a queryset_only attribute set to True are never copied

具体的にどのメソッドがダメなのかよくわからない。誰か助けて。

# まとめ　
Djangoとしてはテーブル水準ではManagerのカスタム、それ以外のSQLの細かいフィルターなどはQuerySetでやってほしいのかな？という感じがした。
QuerySetをas_managerとした方が、重ねて利用しやすい分良い気がする。
Managerを直接カスタムする場合の利点があまりわからなかったので、
もしご存知の人がいたら教えて下さい。

[^1]: 公式の日本語訳だとQuerySetは返してはいけない、となっていましたが
原文はA custom Manager method can return anything you want. It doesn’t have to return a QuerySet.


# 参考文献
https://docs.djangoproject.com/en/4.1/topics/db/managers/#creating-a-manager-with-queryset-methods
https://stackoverflow.com/questions/29798125/when-should-i-use-a-custom-manager-versus-a-custom-queryset-in-django
https://jairvercosa.medium.com/manger-vs-query-sets-in-django-e9af7ed744e0
