.. peewee documentation master file, created by
   sphinx-quickstart on Thu Nov 25 21:20:29 2010.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

peewee
======

.. image:: peewee3-logo.png

Peewee はシンプルで小さな ORM です.概念的な話は(表現豊かながら)多くはなく,
学びやすく直感的に利用できるようにしています.

* 小さくかつ表現豊かな ORM
* python 2.7+ and 3.4+ (developed with 3.6)
* sqlite, mysql, postgresql, cockroachdb をサポート
* :ref:`多数のエクステンション <playhouse>`

.. image:: postgresql.png
    :target: peewee/database.html#using-postgresql
    :alt: postgresql

.. image:: mysql.png
    :target: peewee/database.html#using-mysql
    :alt: mysql

.. image:: sqlite.png
    :target: peewee/database.html#using-sqlite
    :alt: sqlite

.. image:: crdb.png
    :target: peewee/database.html#using-crdb
    :alt: cockroachdb

Peewee のソースコードは `GitHub <https://github.com/coleifer/peewee>`_ でホストされています.

Peewee は初めてですか？以下がお役に立てるかもしれません:

* :ref:`クイックスタート <quickstart>`
* :ref:`ツイッターアプリの例 <example-app>`
* :ref:`peewee を会話的に使う <interactive>`
* :ref:`モデルとフィールド <models>`
* :ref:`クエリーの発行 <querying>`
* :ref:`リレーションとJOIN <relationships>`

Contents:
---------

.. toctree::
   :maxdepth: 2
   :glob:

   peewee/installation
   peewee/quickstart
   peewee/example
   peewee/interactive
   peewee/contributing
   peewee/database
   peewee/models
   peewee/querying
   peewee/query_operators
   peewee/relationships
   peewee/api
   peewee/sqlite_ext
   peewee/playhouse
   peewee/query_examples
   peewee/query_builder
   peewee/hacks
   peewee/changes

Note
----

何かバグや妙な動きを見つけた場合,もしくは新しいアイデアがある場合,GitHubで
`open an issue <https://github.com/coleifer/peewee/issues?state=open>`_ するか、
または `contact me <http://charlesleifer.com/contact/>`_ に連絡してください.

日本語マニュアルに関する誤植や改善提案については,
`open an issue(日本語マニュアル用) <https://github.com/hotta/peewee/issues>`_ 
までお願いします.翻訳後のサイトは 
`https://net-newbie.com/peewee/ <https://net-newbie.com/peewee/>`_ 
にあります.

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
