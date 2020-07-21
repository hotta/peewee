.. _querying:

クエリーの発行（まだ途中）
==========================

この章では, リレーショナルデータベースに対して広く実行される, 基本的な CRUD 操作について述べます:

* :py:meth:`Model.create`, *INSERT* クエリーの発行.
* :py:meth:`Model.save` と :py:meth:`Model.update`,  *UPDATE* クエリーの発行.
* :py:meth:`Model.delete_instance` と :py:meth:`Model.delete`, *DELETE* クエリーの発行.
* :py:meth:`Model.select`, *SELECT* クエリーの発行.

.. note::
    この文書では, `Postgresql Exercises <https://pgexercises.com/>`_ Web
    サイトにある大量のクエリー例を取り入れています.クエリー例の一覧は
    :ref:`query examples <query_examples>` ドキュメントにあります.

新しいレコードを作成する
-------------------------

:py:meth:`Model.create` を使って新しいモデルのインスタンスを作成できます.
このメソッドはキーワード引数を受け取りますが, キーはモデルのフィールド名に対応しています.
新しいインスタンスが返され, テーブルに行が追加されます.

.. code-block:: pycon

    >>> User.create(username='Charlie')
    <__main__.User object at 0x2529350>

これはデータベースに新しい行を *INSERT* します.
プライマリキーが自動的に取り出され, モデルのインスタンスに格納されます.

別案として, プログラム的にモデルインスタンスを組み立てて,
:py:meth:`~Model.save` を呼ぶこともできます:

.. code-block:: pycon

    >>> user = User(username='Charlie')
    >>> user.save()  # save() returns the number of rows modified.
    1
    >>> user.id
    1
    >>> huey = User()
    >>> huey.username = 'Huey'
    >>> huey.save()
    1
    >>> huey.id
    2

モデルが外部キーを保つ場合は, 新しいレコードを作成する際に,
モデルインスタンスを外部キーフィールドに直接代入できます.

.. code-block:: pycon

    >>> tweet = Tweet.create(user=huey, message='Hello!')

関連するオブジェクトのプライマリキーの値を使うこともできます:

.. code-block:: pycon

    >>> tweet = Tweet.create(user=2, message='Hello again!')

もし単にデータを insert したいだけで, モデルインスタンスを作る必要がない場合,
 :py:meth:`Model.insert` が使えます:

.. code-block:: pycon

    >>> User.insert(username='Mickey').execute()
    3

insert クエリーを実行した後, 新しい行のプライマリキーが返されます.

.. note::
    一括 insert の際の速度を上げるための方法がいくつかあります.
    詳細は :ref:`bulk_inserts` の方法の章を確認してみてください.

.. _bulk_inserts:

一括 insert
------------

たくさんのデータを素早くロードするための方法をいくつかご紹介します.
バカ正直なアプローチとしては, 単にループの中で :py:meth:`Model.create` を呼ぶことです:


.. code-block:: python

    data_source = [
        {'field1': 'val1-1', 'field2': 'val1-2'},
        {'field1': 'val2-1', 'field2': 'val2-2'},
        # ...
    ]

    for data_dict in data_source:
        MyModel.create(**data_dict)

上記のアプローチは, いくつかの理由により遅くなります:

1.  ループをトランザクションで囲んでいない場合, :py:meth:`~Model.create`
    への呼び出しのたびにトランザクションが生成されます. これだと極端に遅くなります!
2.  この方法だと, これに 適合する Python のロジックがたくさんあるのですが, それぞれに対して
    :py:class:`InsertQuery` を生成し, それらが SQL にパースされる必要があります.
3.  このため, データベースに対して(SQL の生のバイトストリームという意味で)
    パース対象となる大量のデータを送りつけることになります.
4.  私達は *last insert id* を取り出しますが, 
    このために追加のクエリーを発行しなければならないケースがあります.

これを :py:meth:`~Database.atomic` を使ってトランザクションで囲むだけで, 劇的に速くなります.


.. code-block:: python

    # この方が速くなる.
    with db.atomic():
        for data_dict in data_source:
            MyModel.create(**data_dict)

上記のコードでは, まだ 2,3,4 の弱点があります. :py:meth:`~Model.insert_many` 
を使うと, さらに爆速になります. このメソッドはリストまたは辞書を受け取り,
1回の単独クエリーで複数の行を insert します.

.. code-block:: python

    data_source = [
        {'field1': 'val1-1', 'field2': 'val1-2'},
        {'field1': 'val2-1', 'field2': 'val2-2'},
        # ...
    ]

    # 複数行を INSERT するための, より速いやり方
    MyModel.insert_many(data_source).execute()

:py:meth:`~Model.insert_many` メソッドは行タプルのリストも受け取れるので,
対応するフィールドを指定することもできます:

.. code-block:: python

    # タプルの INSERT はできるが...
    data = [('val1-1', 'val1-2'),
            ('val2-1', 'val2-2'),
            ('val3-1', 'val3-2')]

    # 値がどのフィールドに対応するのかを指定する必要がある.
    MyModel.insert_many(data, fields=[MyModel.field1, MyModel.field2]).execute()

一括 insert をトランザクションで囲むのも好ましいやり方です:

.. code-block:: python

    # もちろんこれをトランザクションで囲むこともできる:
    with db.atomic():
        MyModel.insert_many(data, fields=fields).execute()

.. note::
    SQLite ユーザは一括 insert に際して注意すべき事項があります. 特に SQLite3 のバージョンが
    3.7.11.0 もしくはそれ以降の場合, 一括 insert API が使えるという利点があります. さらに,
    SQLite では SQL クエリー中のバインド変数の数がデフォルトで ``999`` に制限されています.

一括で行を insert する
^^^^^^^^^^^^^^^^^^^^^^^^^

データソース中の行数次第では, それらを複数に分割する必要があるケースがあります.
特に SQLite はクエリーごとの変数が
`999 に制限 <https://www.sqlite.org/limits.html#max_variable_number>`_
されています(バッチのサイズは概ね 1000 / 行の長さ).

1回分のデータを複数のブロックに分割するためのループを書くことができます
(このケースでは, トランザクションを使うことが **強く推奨されます** .

.. code-block:: python

    # 一度に 100 行ずつ insert する
    with db.atomic():
        for idx in range(0, len(data_source), 100):
            MyModel.insert_many(data_source[idx:idx+100]).execute()

Peewee には :py:func:`chunked` ヘルパー関数が用意されており, これを使うと一般的な
iterable(繰り返しループ)を *効率的に* *batch*-size の大きさの iterable に変換できます:


.. code-block:: python

    from peewee import chunked

    # 一度に 100 行ずつ insert する
    with db.atomic():
        for batch in chunked(data_source, 100):
            MyModel.insert_many(batch).execute()

別の方法
^^^^^^^^^^^^

:py:meth:`Model.bulk_create` メソッドは :py:meth:`Model.insert_many` 
とよく似た動作をするのですが, これと違うところは,
未保存の(unsaved)モデルインスタンスのリストを受け取って insert を行い,
またオプションで batch-size パラメータを受け付けるところです.
:py:meth:`~Model.bulk_create` API の使い方は以下のとおりです:

.. code-block:: python

    # 一例として, ファイルからユーザ名のリストを読み込む
    with open('user_list.txt') as fh:
        # 未保存の User インスタンスのリストを作成する
        users = [User(username=line.strip()) for line in fh.readlines()]

    # 操作をトランザクションで囲み, 一度に 100 個ずつ users に insert する
    with db.atomic():
        User.bulk_create(users, batch_size=100)

.. note::
    ( ``RETURNING`` 句をサポートしている) Postgresql をお使いの場合,
    前述の未保存のモデルインスタンスでは, 
    それらの新しいプライマリキーの値が自動的に付与されます.

さらに, Peewee では :py:meth:`Model.bulk_update` を提供しています.
これはモデルのリストにおける１つ以上のカラムを効率的に update します.
以下に例を示します:

.. code-block:: python

    # まず u1, u2, u3 の３つのユーザを作成する
    u1, u2, u3 = [User.create(username='u%s' % i) for i in (1, 2, 3)]

    # 次に user のインスタンスを変更します.
    u1.username = 'u1-x'
    u2.username = 'u2-y'
    u3.username = 'u3-z'

    # ３つのすべての user を一つの update クエリーで update します.
    User.bulk_update([u1, u2, u3], fields=[User.username])

.. note::
    巨大なオブジェクトのリストを扱う場合, 適切な batch_size を指定し, 
    かつ :py:meth:`~Model.bulk_update` の呼び出しを :py:meth:`Database.atomic`
    で囲むようにしてください:

    .. code-block:: python

        with database.atomic():
            User.bulk_update(list_of_users, fields=['username'], batch_size=50)

別の方法として :py:meth:`Database.batch_commit` ヘルパーを使い, *batch*-size
になったトランザクションの中で行ブロック(chunks of rows)を処理することもできます.
このメソッドは, Postgresql 以外のデータベースを使っている場合に,
新しく作られた行のプライマリキーを取得しなければならないケースにおける回避策を提供します.

.. code-block:: python

    # insert する行データのリスト
    row_data = [{'username': 'u1'}, {'username': 'u2'}, ...]

    # row_data には 789 個のデータが入っているとする. 以下のコードでは,
    # 合計８個のトランザクションが発生する(7x100 行 + 1x89 行)
    for row in db.batch_commit(row_data, 100):
        User.create(**row)

他のテーブルからの一括ローディング
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一括ロード対象のデータが別のテーブルに入っている場合, ソースが *SELECT*
クエリーであるような *INSERT* クエリーを作成することもできます.
:py:meth:`Model.insert_from` メソッドを使ってみてください:

.. code-block:: python

    res = (TweetArchive
           .insert_from(
               Tweet.select(Tweet.user, Tweet.message),
               fields=[TweetArchive.user, TweetArchive.message])
           .execute())

上記のクエリーは, 以下の SQL と同じ意味です:

.. code-block:: sql

    INSERT INTO "tweet_archive" ("user_id", "message")
    SELECT "user_id", "message" FROM "tweet";


既存のレコードを update する
-------------------------------

モデルインスタンスがプライマリキーを持っていれば, それ以降の :py:meth:`~Model.save`
へのコールでは, 別レコードの *INSERT* ではなく *UPDATE* が行われるようになります.
そのモデルのプライマリキーは変更されません:

.. code-block:: pycon

    >>> user.save()  # save() は変更された行数を返す
    1
    >>> user.id
    1
    >>> user.save()
    >>> user.id
    1
    >>> huey.save()
    1
    >>> huey.id
    2

複数のレコードを update したい場合は *UPDATE* クエリーを発行します.
以下の例では昨日以前に作成された ``Tweet`` オブジェクトを update してそれらを *published* の状態にします.
:py:meth:`Model.update` はキーワード引数を受け付けますが, その際のキーはモデルのフィールド名に対応します:

.. code-block:: pycon

    >>> today = datetime.today()
    >>> query = Tweet.update(is_published=True).where(Tweet.creation_date < today)
    >>> query.execute()  # Returns the number of rows that were updated.
    4

詳細は :py:meth:`Model.update`, :py:class:`Update`, :py:meth:`Model.bulk_update`
のドキュメントを参照してください.

.. note::
    (カラムの値をインクリメントするといった)アトミックな update の実行に関する詳細情報については,
    :ref:`atomic update <atomic_updates>` レシピを参照してください.

.. _atomic_updates:

アトミックな update
----------------------

Peewee ではアトミックな update を実行できます.
いくつかのカウンターを update する必要があるとしましょう.
ネイティブなアプローチを使う場合は以下のようになるでしょう:

.. code-block:: pycon

    >>> for stat in Stat.select().where(Stat.url == request.url):
    ...     stat.counter += 1
    ...     stat.save()

**これをやってはいけません!** これは遅いだけではなく脆弱であり,
複数のプロセスが同時にカウンターを update しようとしている場合に競合が発生する恐れがあります.

代わりに :py:meth:`~Model.update` を使ってカウンターを自動的に update するようにしましょう:

.. code-block:: pycon

    >>> query = Stat.update(counter=Stat.counter + 1).where(Stat.url == request.url)
    >>> query.execute()

以下のように複雑な update 文を作ることもできます.
従業員へのボーナスを, 前回のボーナス支給額にその人の給与の 10% を上乗せした額としましょう:

.. code-block:: pycon

    >>> query = Employee.update(bonus=(Employee.bonus + (Employee.salary * .1)))
    >>> query.execute()             # みんなにボーナスをやるぞ!

サブクエリーを使ってカラムの値を更新することもできます. ``User``
モデルの中に非正規化されたカラムがあって, そこにはユーザがツイートを行った回数が入っており,
これを定期的に更新することを考えます.これを実現するには以下のようになるでしょう:

.. code-block:: pycon

    >>> subquery = Tweet.select(fn.COUNT(Tweet.id)).where(Tweet.user == User.id)
    >>> update = User.update(num_tweets=subquery)
    >>> update.execute()

Upsert
^^^^^^

Peewee では変則的なタイプである upsert 機能をサポートしています.
SQLite(3.24.0 以前)もしくは MySQL について, Peewee では :py:meth:`~Model.replace`
を提供しており、これはレコードを insert して, その際に制約違反があれば既存のレコードを置き換えます.

:py:meth:`~Model.replace` と :py:meth:`~Insert.on_conflict_replace` の例を示します:

.. code-block:: python

    class User(Model):
        username = TextField(unique=True)
        last_login = DateTimeField(null=True)

    # ユーザを insert または update する. "last_login" の値は
    # そのユーザが既存ユーザであるかどうかを問わずに update される.
    user_id = (User
               .replace(username='the-user', last_login=datetime.now())
               .execute())

    # これも同等の動きをする:
    user_id = (User
               .insert(username='the-user', last_login=datetime.now())
               .on_conflict_replace()
               .execute())

.. note::
    もし insert した際に制約条件が発生したら単に無視したい場合, *replace* に加えて, 
    SQLite, MySQL, Postgresql では *ignore* アクションを提供しています
    ( :py:meth:`~Insert.on_conflict_ignore` を参照).

**MySQL** では *ON DUPLICATE KEY UPDATE* 句を通した upsert をサポートしています.
以下に例を示します:

.. code-block:: python

    class User(Model):
        username = TextField(unique=True)
        last_login = DateTimeField(null=True)
        login_count = IntegerField()

    # 新しいユーザを insert する
    User.create(username='huey', login_count=0)

    # ユーザのログインをシミュレートする. 
    # ログインカウントとタイムスタンプの両方が正しく作成または update される.
    now = datetime.now()
    rowid = (User
             .insert(username='huey', last_login=now, login_count=1)
             .on_conflict(
                 preserve=[User.last_login],  # insert した時の値を使う
                 update={User.login_count: User.login_count + 1})
             .execute())

上記の例を使うと, 必要であれば何度でも upsert クエリーを発行できます.
ログイン回数は自動的にインクリメントされ, last_login カラムは update され,
重複行が発生することがありません.

**Postgresql と SQLite** (3.24.0 以降)では, 別の文法により提供しています.
これは, どの制約違反が競合解決のトリガーとなるべきなのか, およびどの値を更新／保持すべきかを,
より細かい粒度で制御することが可能です.

:py:meth:`~Insert.on_conflict` を使って Postgresql スタイル(もしくは SQLite 3.24+) で
upsert する例を以下に示します:

.. code-block:: python

    class User(Model):
        username = TextField(unique=True)
        last_login = DateTimeField(null=True)
        login_count = IntegerField()

    # 新しいユーザを insert
    User.create(username='huey', login_count=0)

    # ユーザのログインをシミュレートする. 
    # ログインカウントとタイムスタンプの両方が正しく作成または update される.
    now = datetime.now()
    rowid = (User
             .insert(username='huey', last_login=now, login_count=1)
             .on_conflict(
                 conflict_target=[User.username],  # どの制約条件か?
                 preserve=[User.last_login],       # insert した時の値を使う
                 update={User.login_count: User.login_count + 1})
             .execute())

上記の例を使うと, 必要であれば何度でも upsert クエリーを発行できます.
ログイン回数は自動的にインクリメントされ, last_login カラムは update され,
重複行が発生することがありません.

.. note::
    MySQL と Postgresql/SQLite との主な違いとしては, 後者は  ``conflict_target``
    の指定が必要となります.

(もしこれが怪しげに見える場合は) :py:class:`EXCLUDED` 名前空間を使ったより高度な例を示します.
:py:class:`EXCLUDED` ヘルパーを使うと, 競合するデータの中で値を参照できるようになります.
以下の例ではユニークなキー(string)から値(integer)へのマッピングを行うシンプルなテーブルを想定します:

.. code-block:: python

    class KV(Model):
        key = CharField(unique=True)
        value = IntegerField()

    # 1行を作成
    KV.create(key='k1', value=1)

    # EXCLUDED を使ったデモを行います.
    # ここでは指定されたキーで新しい値を insert しようとしています.
    # そのキーがすでに存在する場合, その値を元の値の *合計* で update し,
    # その結果を insert します - 新しい値は元の値より大きくなるはずです.
    query = (KV.insert(key='k1', value=10)
             .on_conflict(conflict_target=[KV.key],
                          update={KV.value: KV.value + EXCLUDED.value},
                          where=(EXCLUDED.value > KV.value)))

    # 上記のクエリーを発行すると, "kv" テーブルで既存のデータが
    # (key='k1', value=11) のようになります:
    query.execute()

    # もしこのクエリーを *もう一度* 実行した場合, 何も更新されません.
    # これは新しい値(10)は元の値(11)より小さいからです.

詳細は :py:meth:`Insert.on_conflict` および :py:class:`OnConflict` を参照してください.

レコードの削除
----------------

単一モデルインスタンスの削除では :py:meth:`Model.delete_instance` ショットカットが使えます.
:py:meth:`~Model.delete_instance` は指定されたモデルインスタンスを削除し,
さらにオプション( `recursive=True` 指定)で これに依存するオブジェクトを再帰的に削除します.

.. code-block:: pycon

    >>> user = User.get(User.id == 1)
    >>> user.delete_instance()          # 削除件数が返される
    1

    >>> User.get(User.id == 1)
    UserDoesNotExist: instance matching query does not exist:
    SQL: SELECT t1."id", t1."username" FROM "user" AS t1 WHERE t1."id" = ?
    PARAMS: [1]

任意の行セットを削除する場合は *DELETE* クエリーを発行してください。
以下の例では1年以上経過した ``Tweet`` オブジェクトを削除します.

.. code-block:: pycon

    >>> query = Tweet.delete().where(Tweet.creation_date < one_year_ago)
    >>> query.execute()                 # 削除件数が返される
    7

詳細は以下のドキュメントを参照してください:

* :py:meth:`Model.delete_instance`
* :py:meth:`Model.delete`
* :py:class:`DeleteQuery`

単一のレコードを select する
---------------------------------

:py:meth:`Model.get` メソッドを使って、指定されたクエリーにマッチする
単一のインスタンスを取り出すことができます. プライマリキーを検索する場合、
:py:meth:`Model.get_by_id` というショートカットメソッドを使うことも
できます。

このメソッドは、指定されたクエリーを使って :py:meth:`Model.select` を呼ぶ
ことへのショートカットです。さらに、指定されたクエリーにマッチするモデルが
なかった場合、 ``DoesNotExist`` 例外が送出されます。

.. code-block:: pycon

    >>> User.get(User.id == 1)
    <__main__.User object at 0x25294d0>

    >>> User.get_by_id(1)  # 上と同じ.
    <__main__.User object at 0x252df10>

    >>> User[1]  # これも上と同じ.
    <__main__.User object at 0x252dd10>

    >>> User.get(User.id == 1).username
    u'Charlie'

    >>> User.get(User.username == 'Charlie')
    <__main__.User object at 0x2529410>

    >>> User.get(User.username == 'nobody')
    UserDoesNotExist: instance matching query does not exist:
    SQL: SELECT t1."id", t1."username" FROM "user" AS t1 WHERE t1."username" = ?
    PARAMS: ['nobody']

さらに高度な操作を行いたい場合、 :py:meth:`SelectBase.get` が使えます。
以下のクエリーでは *charlie* という名前のユーザからの、最新のツイートを
取り出しています。:

.. code-block:: pycon

    >>> (Tweet
    ...  .select()
    ...  .join(User)
    ...  .where(User.username == 'charlie')
    ...  .order_by(Tweet.created_date.desc())
    ...  .get())
    <__main__.Tweet object at 0x2623410>

詳細は以下のドキュメントを参照してください:

* :py:meth:`Model.get`
* :py:meth:`Model.get_by_id`
* :py:meth:`Model.get_or_none` - if no matching row is found, return ``None``.
* :py:meth:`Model.first`
* :py:meth:`Model.select`
* :py:meth:`SelectBase.get`

あれば get なければ create
-------------------------------

Peewee では get/create タイプの操作を実行するヘルパーメソッド:
:py:meth:`Model.get_or_create` を備えています。これは、まずマッチする行を
取り出そうとします。これに失敗すると、新しい行が作られます。

"create または get" タイプのロジックにおいては、一般的に *unique* 制約
もしくはプライマリキーにより、重複したオブジェクトを作るのを防いでいます。
一例として、ここでは :ref:`example User model <blog-models>` を使って
新しいユーザーアカウントを登録するための実装をしたいものとします。
*User* モデルは username フィールドについて *unique* 制約を持っているため、
私達はデータベースの整合性保証の枠組みに依存することで、重複した username
を生成してしまうこと防げます:

.. code-block:: python

    try:
        with db.atomic():
            return User.create(username=username)
    except peewee.IntegrityError:
        # `username` はユニークなカラムなので、username がすでに存在
        # する場合、安全に .get() の呼び出しを行える。
        return User.get(User.username == username)

このような種類のロジックを、あなたの ``Model`` クラスの ``classmethod`` として、
容易にカプセル化できます。

前述の例ではまず生成を試み、それが失敗したら取得へとフォールバックしますが、
これはデータベースの unique 制約に依存します。もし、まずレコードの取得を
試みたいという場合は :py:meth:`~Model.get_or_create` が使えます。この
メソッドは Django の同名の関数と同じように実装されています。 フィルターとして
``WHERE`` 条件を指定する場合も Django スタイルのキーワード引数が使えます。
この関数は、インスタンス自身、およびオブジェクトが作られたかどうかを表す 
boolean 値からなる２要素のタプルを返します。

:py:meth:`~Model.get_or_create` を使ってユーザーアカウントの作成処理を
実装する方法は以下の通りです:

.. code-block:: python

    user, created = User.get_or_create(username=username)

さて、ここで別の ``Person`` モデルがあり、これを使ってオブジェクトの取得または
生成を行いたいとします。 ``Person`` の取得にあたって必要な条件は彼らの姓と名
だけなのです **が、しかし** 新しいレコードを作る際には結局彼らの生年月日や
好きな色なども指定することになります:

.. code-block:: python

    person, created = Person.get_or_create(
        first_name=first_name,
        last_name=last_name,
        defaults={'dob': dob, 'favorite_color': 'green'})

:py:meth:`~Model.get_or_create` に渡されたキーワード引数は、 ``defaults``
辞書を除き、すべてロジックの ``get()`` 部分で使われます。 ``defaults``
部分は新しく生成されたインスタンスで値を展開するのに使われます。

詳細は :py:meth:`Model.get_or_create` のドキュメントを参照してください。

複数レコードの select
--------------------------

:py:meth:`Model.select` を使ってテーブルから行を取り出せます。 *SELECT* 
クエリーを構築する際、データベースはあなたのクエリーに該当する行を返します。
Peewee ではインデックスやスライス操作を使うだけでなく、これらの行からの
イテレートもできます:

.. code-block:: pycon

    >>> query = User.select()
    >>> [user.username for user in query]
    ['Charlie', 'Huey', 'Peewee']

    >>> query[1]
    <__main__.User at 0x7f83e80f5550>

    >>> query[1].username
    'Huey'

    >>> query[:2]
    [<__main__.User at 0x7f83e80f53a8>, <__main__.User at 0x7f83e80f5550>]

:py:class:`Select` クエリーは賢いので、この中でイテレートやインデックスによる
アクセスやスライスを何度行っても、実際にクエリーが実行されるのは一度だけです。

以下の例では単に :py:meth:`~Model.select` へのコールを行い、その戻り値である
:py:class:`Select` のインスタンスに対してイテレートを行います。これは *User*
テーブルの中のすべての行を返します:

.. code-block:: pycon

    >>> for user in User.select():
    ...     print user.username
    ...
    Charlie
    Huey
    Peewee

.. note::
    同一クエリーに対する後続のイテレートは、クエリーの結果がキャッシュされて
    いるためデータベースにはヒットしません。この振る舞いを無効にする（メモリ
    の使用量を減らす）には、イテレートの際に :py:meth:`Select.iterator` を
    コールしてください。

外部キーを持つモデルに対してイテレートする場合、関連するモデルの値へのアクセス
には注意してください。外部キーまたは後方参照に対するイテレートは、意図しない
:ref:`N+1 query behavior <nplusone>` を起こす恐れがあります。

``Tweet.user`` のような外部キーを作成する場合、 *backref* を使って
(``User.tweets``) という後方参照を作成できます。後方参照は :py:class:`Select`
インスタンスとして露出されます:

.. code-block:: pycon

    >>> tweet = Tweet.get()
    >>> tweet.user  # 関連するモデルを返すような外部キーへのアクセス
    <tw.User at 0x7f3ceb017f50>

    >>> user = User.get()
    >>> user.tweets  # クエリーを返す後方参照へのアクセス
    <peewee.ModelSelect at 0x7f73db3bafd0>

他の :py:class:`Select` と同様に、``user.tweets`` 後方参照を通したイテレートが
可能です:

.. code-block:: pycon

    >>> for tweet in user.tweets:
    ...     print(tweet.message)
    ...
    hello world
    this is fun
    look at this picture of my food

モデルインスタンスを返すだけでなく、 :py:class:`Select` クエリーは辞書やタプル、
および名前付きタプルを返すことが可能です。ご自分のユースケースにもよりますが、
行を辞書として扱うほうが簡単な場合もあります。以下に例を示します:

.. code-block:: pycon

    >>> query = User.select().dicts()
    >>> for row in query:
    ...     print(row)

    {'id': 1, 'username': 'Charlie'}
    {'id': 2, 'username': 'Huey'}
    {'id': 3, 'username': 'Peewee'}

詳細は :py:meth:`~BaseQuery.namedtuples`, :py:meth:`~BaseQuery.tuples`,
:py:meth:`~BaseQuery.dicts` を参照してください。

巨大な結果セットをイテレートする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default peewee will cache the rows returned when iterating over a
:py:class:`Select` query. This is an optimization to allow multiple iterations
as well as indexing and slicing without causing additional queries. This
caching can be problematic, however, when you plan to iterate over a large
number of rows.

To reduce the amount of memory used by peewee when iterating over a query, use
the :py:meth:`~BaseQuery.iterator` method. This method allows you to iterate
without caching each model returned, using much less memory when iterating over
large result sets.

.. code-block:: python

    # Let's assume we've got 10 million stat objects to dump to a csv file.
    stats = Stat.select()

    # Our imaginary serializer class
    serializer = CSVSerializer()

    # Loop over all the stats and serialize.
    for stat in stats.iterator():
        serializer.serialize_object(stat)

For simple queries you can see further speed improvements by returning rows as
dictionaries, namedtuples or tuples. The following methods can be used on any
:py:class:`Select` query to change the result row type:

* :py:meth:`~BaseQuery.dicts`
* :py:meth:`~BaseQuery.namedtuples`
* :py:meth:`~BaseQuery.tuples`

Don't forget to append the :py:meth:`~BaseQuery.iterator` method call to also
reduce memory consumption. For example, the above code might look like:

.. code-block:: python

    # Let's assume we've got 10 million stat objects to dump to a csv file.
    stats = Stat.select()

    # Our imaginary serializer class
    serializer = CSVSerializer()

    # Loop over all the stats (rendered as tuples, without caching) and serialize.
    for stat_tuple in stats.tuples().iterator():
        serializer.serialize_tuple(stat_tuple)

When iterating over a large number of rows that contain columns from multiple
tables, peewee will reconstruct the model graph for each row returned. This
operation can be slow for complex graphs. For example, if we were selecting a
list of tweets along with the username and avatar of the tweet's author, Peewee
would have to create two objects for each row (a tweet and a user). In addition
to the above row-types, there is a fourth method :py:meth:`~BaseQuery.objects`
which will return the rows as model instances, but will not attempt to resolve
the model graph.

For example:

.. code-block:: python

    query = (Tweet
             .select(Tweet, User)  # Select tweet and user data.
             .join(User))

    # Note that the user columns are stored in a separate User instance
    # accessible at tweet.user:
    for tweet in query:
        print(tweet.user.username, tweet.content)

    # Using ".objects()" will not create the tweet.user object and assigns all
    # user attributes to the tweet instance:
    for tweet in query.objects():
        print(tweet.username, tweet.content)

For maximum performance, you can execute queries and then iterate over the
results using the underlying database cursor. :py:meth:`Database.execute`
accepts a query object, executes the query, and returns a DB-API 2.0 ``Cursor``
object. The cursor will return the raw row-tuples:

.. code-block:: python

    query = Tweet.select(Tweet.content, User.username).join(User)
    cursor = database.execute(query)
    for (content, username) in cursor:
        print(username, '->', content)

レコードのフィルタリング
--------------------------

You can filter for particular records using normal python operators. Peewee
supports a wide variety of :ref:`query operators <query-operators>`.

.. code-block:: pycon

    >>> user = User.get(User.username == 'Charlie')
    >>> for tweet in Tweet.select().where(Tweet.user == user, Tweet.is_published == True):
    ...     print(tweet.user.username, '->', tweet.message)
    ...
    Charlie -> hello world
    Charlie -> this is fun

    >>> for tweet in Tweet.select().where(Tweet.created_date < datetime.datetime(2011, 1, 1)):
    ...     print(tweet.message, tweet.created_date)
    ...
    Really old tweet 2010-01-01 00:00:00

You can also filter across joins:

.. code-block:: pycon

    >>> for tweet in Tweet.select().join(User).where(User.username == 'Charlie'):
    ...     print(tweet.message)
    hello world
    this is fun
    look at this picture of my food

If you want to express a complex query, use parentheses and python's bitwise
*or* and *and* operators:

.. code-block:: pycon

    >>> Tweet.select().join(User).where(
    ...     (User.username == 'Charlie') |
    ...     (User.username == 'Peewee Herman'))

.. note::
    Note that Peewee uses **bitwise** operators (``&`` and ``|``) rather than
    logical operators (``and`` and ``or``). The reason for this is that Python
    coerces the return value of logical operations to a boolean value. This is
    also the reason why "IN" queries must be expressed using ``.in_()`` rather
    than the ``in`` operator.

Check out :ref:`the table of query operations <query-operators>` to see what
types of queries are possible.

.. note::

    A lot of fun things can go in the where clause of a query, such as:

    * A field expression, e.g. ``User.username == 'Charlie'``
    * A function expression, e.g. ``fn.Lower(fn.Substr(User.username, 1, 1)) == 'a'``
    * A comparison of one column to another, e.g. ``Employee.salary < (Employee.tenure * 1000) + 40000``

    You can also nest queries, for example tweets by users whose username
    starts with "a":

    .. code-block:: python

        # get users whose username starts with "a"
        a_users = User.select().where(fn.Lower(fn.Substr(User.username, 1, 1)) == 'a')

        # the ".in_()" method signifies an "IN" query
        a_user_tweets = Tweet.select().where(Tweet.user.in_(a_users))

さらなるクエリーの例
^^^^^^^^^^^^^^^^^^^^^^^^

.. note::
    For a wide range of example queries, see the :ref:`Query Examples <query_examples>`
    document, which shows how to implements queries from the `PostgreSQL Exercises <https://pgexercises.com/>`_
    website.

Get active users:

.. code-block:: python

    User.select().where(User.active == True)

Get users who are either staff or superusers:

.. code-block:: python

    User.select().where(
        (User.is_staff == True) | (User.is_superuser == True))

Get tweets by user named "charlie":

.. code-block:: python

    Tweet.select().join(User).where(User.username == 'charlie')

Get tweets by staff or superusers (assumes FK relationship):

.. code-block:: python

    Tweet.select().join(User).where(
        (User.is_staff == True) | (User.is_superuser == True))

Get tweets by staff or superusers using a subquery:

.. code-block:: python

    staff_super = User.select(User.id).where(
        (User.is_staff == True) | (User.is_superuser == True))
    Tweet.select().where(Tweet.user.in_(staff_super))

レコードのソート
-------------------

To return rows in order, use the :py:meth:`~Query.order_by` method:

.. code-block:: pycon

    >>> for t in Tweet.select().order_by(Tweet.created_date):
    ...     print(t.pub_date)
    ...
    2010-01-01 00:00:00
    2011-06-07 14:08:48
    2011-06-07 14:12:57

    >>> for t in Tweet.select().order_by(Tweet.created_date.desc()):
    ...     print(t.pub_date)
    ...
    2011-06-07 14:12:57
    2011-06-07 14:08:48
    2010-01-01 00:00:00

You can also use ``+`` and ``-`` prefix operators to indicate ordering:

.. code-block:: python

    # The following queries are equivalent:
    Tweet.select().order_by(Tweet.created_date.desc())

    Tweet.select().order_by(-Tweet.created_date)  # Note the "-" prefix.

    # Similarly you can use "+" to indicate ascending order, though ascending
    # is the default when no ordering is otherwise specified.
    User.select().order_by(+User.username)

You can also order across joins. Assuming you want to order tweets by the
username of the author, then by created_date:

.. code-block:: pycon

    query = (Tweet
             .select()
             .join(User)
             .order_by(User.username, Tweet.created_date.desc()))

.. code-block:: sql

    SELECT t1."id", t1."user_id", t1."message", t1."is_published", t1."created_date"
    FROM "tweet" AS t1
    INNER JOIN "user" AS t2
      ON t1."user_id" = t2."id"
    ORDER BY t2."username", t1."created_date" DESC

When sorting on a calculated value, you can either include the necessary SQL
expressions, or reference the alias assigned to the value. Here are two
examples illustrating these methods:

.. code-block:: python

    # Let's start with our base query. We want to get all usernames and the number of
    # tweets they've made. We wish to sort this list from users with most tweets to
    # users with fewest tweets.
    query = (User
             .select(User.username, fn.COUNT(Tweet.id).alias('num_tweets'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User.username))

You can order using the same COUNT expression used in the ``select`` clause. In
the example below we are ordering by the ``COUNT()`` of tweet ids descending:

.. code-block:: python

    query = (User
             .select(User.username, fn.COUNT(Tweet.id).alias('num_tweets'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User.username)
             .order_by(fn.COUNT(Tweet.id).desc()))

Alternatively, you can reference the alias assigned to the calculated value in
the ``select`` clause. This method has the benefit of being a bit easier to
read. Note that we are not referring to the named alias directly, but are
wrapping it using the :py:class:`SQL` helper:

.. code-block:: python

    query = (User
             .select(User.username, fn.COUNT(Tweet.id).alias('num_tweets'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User.username)
             .order_by(SQL('num_tweets').desc()))

Or, to do things the "peewee" way:

.. code-block:: python

    ntweets = fn.COUNT(Tweet.id)
    query = (User
             .select(User.username, ntweets.alias('num_tweets'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User.username)
             .order_by(ntweets.desc())

ランダムなレコードの取得
---------------------------

Occasionally you may want to pull a random record from the database. You can
accomplish this by ordering by the *random* or *rand* function (depending on
your database):

Postgresql and Sqlite use the *Random* function:

.. code-block:: python

    # Pick 5 lucky winners:
    LotteryNumber.select().order_by(fn.Random()).limit(5)

MySQL uses *Rand*:

.. code-block:: python

    # Pick 5 lucky winners:
    LotterNumber.select().order_by(fn.Rand()).limit(5)

レコードのページ制御
-----------------------------

The :py:meth:`~Query.paginate` method makes it easy to grab a *page* or
records. :py:meth:`~Query.paginate` takes two parameters,
``page_number``, and ``items_per_page``.

.. attention::
    Page numbers are 1-based, so the first page of results will be page 1.

.. code-block:: pycon

    >>> for tweet in Tweet.select().order_by(Tweet.id).paginate(2, 10):
    ...     print(tweet.message)
    ...
    tweet 10
    tweet 11
    tweet 12
    tweet 13
    tweet 14
    tweet 15
    tweet 16
    tweet 17
    tweet 18
    tweet 19

If you would like more granular control, you can always use
:py:meth:`~Query.limit` and :py:meth:`~Query.offset`.

レコードのカウント
---------------------

You can count the number of rows in any select query:

.. code-block:: python

    >>> Tweet.select().count()
    100
    >>> Tweet.select().where(Tweet.id > 50).count()
    50

Peewee will wrap your query in an outer query that performs a count, which
results in SQL like:

.. code-block:: sql

    SELECT COUNT(1) FROM ( ... your query ... );

レコードを集約する
-------------------

Suppose you have some users and want to get a list of them along with the count
of tweets in each.

.. code-block:: python

    query = (User
             .select(User, fn.Count(Tweet.id).alias('count'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User))

The resulting query will return *User* objects with all their normal attributes
plus an additional attribute *count* which will contain the count of tweets for
each user. We use a left outer join to include users who have no tweets.

Let's assume you have a tagging application and want to find tags that have a
certain number of related objects. For this example we'll use some different
models in a :ref:`many-to-many <manytomany>` configuration:

.. code-block:: python

    class Photo(Model):
        image = CharField()

    class Tag(Model):
        name = CharField()

    class PhotoTag(Model):
        photo = ForeignKeyField(Photo)
        tag = ForeignKeyField(Tag)

Now say we want to find tags that have at least 5 photos associated with them:

.. code-block:: python

    query = (Tag
             .select()
             .join(PhotoTag)
             .join(Photo)
             .group_by(Tag)
             .having(fn.Count(Photo.id) > 5))

This query is equivalent to the following SQL:

.. code-block:: sql

    SELECT t1."id", t1."name"
    FROM "tag" AS t1
    INNER JOIN "phototag" AS t2 ON t1."id" = t2."tag_id"
    INNER JOIN "photo" AS t3 ON t2."photo_id" = t3."id"
    GROUP BY t1."id", t1."name"
    HAVING Count(t3."id") > 5

Suppose we want to grab the associated count and store it on the tag:

.. code-block:: python

    query = (Tag
             .select(Tag, fn.Count(Photo.id).alias('count'))
             .join(PhotoTag)
             .join(Photo)
             .group_by(Tag)
             .having(fn.Count(Photo.id) > 5))

スカラー値を取り出す
------------------------

You can retrieve scalar values by calling :py:meth:`Query.scalar`. For
instance:

.. code-block:: python

    >>> PageView.select(fn.Count(fn.Distinct(PageView.url))).scalar()
    100

You can retrieve multiple scalar values by passing ``as_tuple=True``:

.. code-block:: python

    >>> Employee.select(
    ...     fn.Min(Employee.salary), fn.Max(Employee.salary)
    ... ).scalar(as_tuple=True)
    (30000, 50000)

.. _window-functions:

Window 関数
----------------

A :py:class:`Window` function refers to an aggregate function that operates on
a sliding window of data that is being processed as part of a ``SELECT`` query.
Window functions make it possible to do things like:

1. Perform aggregations against subsets of a result-set.
2. Calculate a running total.
3. Rank results.
4. Compare a row value to a value in the preceding (or succeeding!) row(s).

peewee comes with support for SQL window functions, which can be created by
calling :py:meth:`Function.over` and passing in your partitioning or ordering
parameters.

For the following examples, we'll use the following model and sample data:

.. code-block:: python

    class Sample(Model):
        counter = IntegerField()
        value = FloatField()

    data = [(1, 10),
            (1, 20),
            (2, 1),
            (2, 3),
            (3, 100)]
    Sample.insert_many(data, fields=[Sample.counter, Sample.value]).execute()

Our sample table now contains:

=== ======== ======
id  counter  value
=== ======== ======
1   1        10.0
2   1        20.0
3   2        1.0
4   2        3.0
5   3        100.0
=== ======== ======

ソートされたウィンドウ
^^^^^^^^^^^^^^^^^^^^^^^^

Let's calculate a running sum of the ``value`` field. In order for it to be a
"running" sum, we need it to be ordered, so we'll order with respect to the
Sample's ``id`` field:

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(order_by=[Sample.id]).alias('total'))

    for sample in query:
        print(sample.counter, sample.value, sample.total)

    # 1    10.    10.
    # 1    20.    30.
    # 2     1.    31.
    # 2     3.    34.
    # 3   100    134.

For another example, we'll calculate the difference between the current value
and the previous value, when ordered by the ``id``:

.. code-block:: python

    difference = Sample.value - fn.LAG(Sample.value, 1).over(order_by=[Sample.id])
    query = Sample.select(
        Sample.counter,
        Sample.value,
        difference.alias('diff'))

    for sample in query:
        print(sample.counter, sample.value, sample.diff)

    # 1    10.   NULL
    # 1    20.    10.  -- (20 - 10)
    # 2     1.   -19.  -- (1 - 20)
    # 2     3.     2.  -- (3 - 1)
    # 3   100     97.  -- (100 - 3)

パーティションされたウィンドウ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Let's calculate the average ``value`` for each distinct "counter" value. Notice
that there are three possible values for the ``counter`` field (1, 2, and 3).
We can do this by calculating the ``AVG()`` of the ``value`` column over a
window that is partitioned depending on the ``counter`` field:

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.AVG(Sample.value).over(partition_by=[Sample.counter]).alias('cavg'))

    for sample in query:
        print(sample.counter, sample.value, sample.cavg)

    # 1    10.    15.
    # 1    20.    15.
    # 2     1.     2.
    # 2     3.     2.
    # 3   100    100.

We can use ordering within partitions by specifying both the ``order_by`` and
``partition_by`` parameters. For an example, let's rank the samples by value
within each distinct ``counter`` group.

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.RANK().over(
            order_by=[Sample.value],
            partition_by=[Sample.counter]).alias('rank'))

    for sample in query:
        print(sample.counter, sample.value, sample.rank)

    # 1    10.    1
    # 1    20.    2
    # 2     1.    1
    # 2     3.    2
    # 3   100     1

境界のあるウィンドウ
^^^^^^^^^^^^^^^^^^^^^^

By default, window functions are evaluated using an *unbounded preceding* start
for the window, and the *current row* as the end. We can change the bounds of
the window our aggregate functions operate on by specifying a ``start`` and/or
``end`` in the call to :py:meth:`Function.over`. Additionally, Peewee comes
with helper-methods on the :py:class:`Window` object for generating the
appropriate boundary references:

* :py:attr:`Window.CURRENT_ROW` - attribute that references the current row.
* :py:meth:`Window.preceding` - specify number of row(s) preceding, or omit
  number to indicate **all** preceding rows.
* :py:meth:`Window.following` - specify number of row(s) following, or omit
  number to indicate **all** following rows.

To examine how boundaries work, we'll calculate a running total of the
``value`` column, ordered with respect to ``id``, **but** we'll only look the
running total of the current row and it's two preceding rows:

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(
            order_by=[Sample.id],
            start=Window.preceding(2),
            end=Window.CURRENT_ROW).alias('rsum'))

    for sample in query:
        print(sample.counter, sample.value, sample.rsum)

    # 1    10.    10.
    # 1    20.    30.  -- (20 + 10)
    # 2     1.    31.  -- (1 + 20 + 10)
    # 2     3.    24.  -- (3 + 1 + 20)
    # 3   100    104.  -- (100 + 3 + 1)

.. note::
    Technically we did not need to specify the ``end=Window.CURRENT`` because
    that is the default. It was shown in the example for demonstration.

Let's look at another example. In this example we will calculate the "opposite"
of a running total, in which the total sum of all values is decreased by the
value of the samples, ordered by ``id``. To accomplish this, we'll calculate
the sum from the current row to the last row.

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(
            order_by=[Sample.id],
            start=Window.CURRENT_ROW,
            end=Window.following()).alias('rsum'))

    # 1    10.   134.  -- (10 + 20 + 1 + 3 + 100)
    # 1    20.   124.  -- (20 + 1 + 3 + 100)
    # 2     1.   104.  -- (1 + 3 + 100)
    # 2     3.   103.  -- (3 + 100)
    # 3   100    100.  -- (100)

フィルターされた集約
^^^^^^^^^^^^^^^^^^^^^^^^^^

Aggregate functions may also support filter functions (Postgres and Sqlite
3.25+), which get translated into a ``FILTER (WHERE...)`` clause. Filter
expressions are added to an aggregate function with the
:py:meth:`Function.filter` method.

For an example, we will calculate the running sum of the ``value`` field with
respect to the ``id``, but we will filter-out any samples whose ``counter=2``.

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).filter(Sample.counter != 2).over(
            order_by=[Sample.id]).alias('csum'))

    for sample in query:
        print(sample.counter, sample.value, sample.csum)

    # 1    10.    10.
    # 1    20.    30.
    # 2     1.    30.
    # 2     3.    30.
    # 3   100    130.

.. note::
    The call to :py:meth:`~Function.filter` must precede the call to
    :py:meth:`~Function.over`.

ウィンドウ定義の再利用
^^^^^^^^^^^^^^^^^^^^^^^^^^

If you intend to use the same window definition for multiple aggregates, you
can create a :py:class:`Window` object. The :py:class:`Window` object takes the
same parameters as :py:meth:`Function.over`, and can be passed to the
``over()`` method in-place of the individual parameters.

Here we'll declare a single window, ordered with respect to the sample ``id``,
and call several window functions using that window definition:

.. code-block:: python

    win = Window(order_by=[Sample.id])
    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.LEAD(Sample.value).over(win),
        fn.LAG(Sample.value).over(win),
        fn.SUM(Sample.value).over(win)
    ).window(win)  # Include our window definition in query.

    for row in query.tuples():
        print(row)

    # counter  value  lead()  lag()  sum()
    # 1          10.     20.   NULL    10.
    # 1          20.      1.    10.    30.
    # 2           1.      3.    20.    31.
    # 2           3.    100.     1.    34.
    # 3         100.    NULL     3.   134.

複数のウィンドウ定義
^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the previous example, we saw how to declare a :py:class:`Window` definition
and re-use it for multiple different aggregations. You can include as many
window definitions as you need in your queries, but it is necessary to ensure
each window has a unique alias:

.. code-block:: python

    w1 = Window(order_by=[Sample.id]).alias('w1')
    w2 = Window(partition_by=[Sample.counter]).alias('w2')
    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(w1).alias('rsum'),  # Running total.
        fn.AVG(Sample.value).over(w2).alias('cavg')   # Avg per category.
    ).window(w1, w2)  # Include our window definitions.

    for sample in query:
        print(sample.counter, sample.value, sample.rsum, sample.cavg)

    # counter  value   rsum     cavg
    # 1          10.     10.     15.
    # 1          20.     30.     15.
    # 2           1.     31.      2.
    # 2           3.     34.      2.
    # 3         100     134.    100.

Similarly, if you have multiple window definitions that share similar
definitions, it is possible to extend a previously-defined window definition.
For example, here we will be partitioning the data-set by the counter value, so
we'll be doing our aggregations with respect to the counter. Then we'll define
a second window that extends this partitioning, and adds an ordering clause:

.. code-block:: python

    w1 = Window(partition_by=[Sample.counter]).alias('w1')

    # By extending w1, this window definition will also be partitioned
    # by "counter".
    w2 = Window(extends=w1, order_by=[Sample.value.desc()]).alias('w2')

    query = (Sample
             .select(Sample.counter, Sample.value,
                     fn.SUM(Sample.value).over(w1).alias('group_sum'),
                     fn.RANK().over(w2).alias('revrank'))
             .window(w1, w2)
             .order_by(Sample.id))

    for sample in query:
        print(sample.counter, sample.value, sample.group_sum, sample.revrank)

    # counter  value   group_sum   revrank
    # 1        10.     30.         2
    # 1        20.     30.         1
    # 2        1.      4.          2
    # 2        3.      4.          1
    # 3        100.    100.        1

.. _window-frame-types:

フレームタイプ: RANGE vs ROWS vs GROUPS
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Depending on the frame type, the database will process ordered groups
differently. Let's create two additional ``Sample`` rows to visualize the
difference:

.. code-block:: pycon

    >>> Sample.create(counter=1, value=20.)
    <Sample 6>
    >>> Sample.create(counter=2, value=1.)
    <Sample 7>

Our table now contains:

=== ======== ======
id  counter  value
=== ======== ======
1   1        10.0
2   1        20.0
3   2        1.0
4   2        3.0
5   3        100.0
6   1        20.0
7   2        1.0
=== ======== ======

Let's examine the difference by calculating a "running sum" of the samples,
ordered with respect to the ``counter`` and ``value`` fields. To specify the
frame type, we can use either:

* :py:attr:`Window.RANGE`
* :py:attr:`Window.ROWS`
* :py:attr:`Window.GROUPS`

The behavior of :py:attr:`~Window.RANGE`, when there are logical duplicates,
may lead to unexpected results:

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(
            order_by=[Sample.counter, Sample.value],
            frame_type=Window.RANGE).alias('rsum'))

    for sample in query.order_by(Sample.counter, Sample.value):
        print(sample.counter, sample.value, sample.rsum)

    # counter  value   rsum
    # 1          10.     10.
    # 1          20.     50.
    # 1          20.     50.
    # 2           1.     52.
    # 2           1.     52.
    # 2           3.     55.
    # 3         100     155.

With the inclusion of the new rows we now have some rows that have duplicate
``category`` and ``value`` values. The :py:attr:`~Window.RANGE` frame type
causes these duplicates to be evaluated together rather than separately.

The more expected result can be achieved by using :py:attr:`~Window.ROWS` as
the frame-type:

.. code-block:: python

    query = Sample.select(
        Sample.counter,
        Sample.value,
        fn.SUM(Sample.value).over(
            order_by=[Sample.counter, Sample.value],
            frame_type=Window.ROWS).alias('rsum'))

    for sample in query.order_by(Sample.counter, Sample.value):
        print(sample.counter, sample.value, sample.rsum)

    # counter  value   rsum
    # 1          10.     10.
    # 1          20.     30.
    # 1          20.     50.
    # 2           1.     51.
    # 2           1.     52.
    # 2           3.     55.
    # 3         100     155.

Peewee uses these rules for determining what frame-type to use:

* If the user specifies a ``frame_type``, that frame type will be used.
* If ``start`` and/or ``end`` boundaries are specified Peewee will default to
  using ``ROWS``.
* If the user did not specify frame type or start/end boundaries, Peewee will
  use the database default, which is ``RANGE``.

The :py:attr:`Window.GROUPS` frame type looks at the window range specification
in terms of groups of rows, based on the ordering term(s). Using ``GROUPS``, we
can define the frame so it covers distinct groupings of rows. Let's look at an
example:

.. code-block:: python

    query = (Sample
             .select(Sample.counter, Sample.value,
                     fn.SUM(Sample.value).over(
                        order_by=[Sample.counter, Sample.value],
                        frame_type=Window.GROUPS,
                        start=Window.preceding(1)).alias('gsum'))
             .order_by(Sample.counter, Sample.value))

    for sample in query:
        print(sample.counter, sample.value, sample.gsum)

    #  counter   value    gsum
    #  1         10       10
    #  1         20       50
    #  1         20       50   (10) + (20+0)
    #  2         1        42
    #  2         1        42   (20+20) + (1+1)
    #  2         3        5    (1+1) + 3
    #  3         100      103  (3) + 100

As you can hopefully infer, the window is grouped by its ordering term, which
is ``(counter, value)``. We are looking at a window that extends between one
previous group and the current group.

.. note::
    For information about the window function APIs, see:

    * :py:meth:`Function.over`
    * :py:meth:`Function.filter`
    * :py:class:`Window`

    For general information on window functions, read the postgres `window functions tutorial <https://www.postgresql.org/docs/current/tutorial-window.html>`_

    Additionally, the `postgres docs <https://www.postgresql.org/docs/current/sql-select.html#SQL-WINDOW>`_
    and the `sqlite docs <https://www.sqlite.org/windowfunctions.html>`_
    contain a lot of good information.

.. _rowtypes:

行タプル／辞書／名前付きタプルの取り出し
--------------------------------------------------

Sometimes you do not need the overhead of creating model instances and simply
want to iterate over the row data without needing all the APIs provided
:py:class:`Model`. To do this, use:

* :py:meth:`~BaseQuery.dicts`
* :py:meth:`~BaseQuery.namedtuples`
* :py:meth:`~BaseQuery.tuples`
* :py:meth:`~BaseQuery.objects` -- accepts an arbitrary constructor function
  which is called with the row tuple.

.. code-block:: python

    stats = (Stat
             .select(Stat.url, fn.Count(Stat.url))
             .group_by(Stat.url)
             .tuples())

    # iterate over a list of 2-tuples containing the url and count
    for stat_url, stat_count in stats:
        print(stat_url, stat_count)

Similarly, you can return the rows from the cursor as dictionaries using
:py:meth:`~BaseQuery.dicts`:

.. code-block:: python

    stats = (Stat
             .select(Stat.url, fn.Count(Stat.url).alias('ct'))
             .group_by(Stat.url)
             .dicts())

    # iterate over a list of 2-tuples containing the url and count
    for stat in stats:
        print(stat['url'], stat['ct'])

.. _returning-clause:

Returning 句
----------------

:py:class:`PostgresqlDatabase` supports a ``RETURNING`` clause on ``UPDATE``,
``INSERT`` and ``DELETE`` queries. Specifying a ``RETURNING`` clause allows you
to iterate over the rows accessed by the query.

By default, the return values upon execution of the different queries are:

* ``INSERT`` - auto-incrementing primary key value of the newly-inserted row.
  When not using an auto-incrementing primary key, Postgres will return the new
  row's primary key, but SQLite and MySQL will not.
* ``UPDATE`` - number of rows modified
* ``DELETE`` - number of rows deleted

When a returning clause is used the return value upon executing a query will be
an iterable cursor object.

Postgresql allows, via the ``RETURNING`` clause, to return data from the rows
inserted or modified by a query.

For example, let's say you have an :py:class:`Update` that deactivates all
user accounts whose registration has expired. After deactivating them, you want
to send each user an email letting them know their account was deactivated.
Rather than writing two queries, a ``SELECT`` and an ``UPDATE``, you can do
this in a single ``UPDATE`` query with a ``RETURNING`` clause:

.. code-block:: python

    query = (User
             .update(is_active=False)
             .where(User.registration_expired == True)
             .returning(User))

    # Send an email to every user that was deactivated.
    for deactivate_user in query.execute():
        send_deactivation_email(deactivated_user.email)

The ``RETURNING`` clause is also available on :py:class:`Insert` and
:py:class:`Delete`. When used with ``INSERT``, the newly-created rows will be
returned. When used with ``DELETE``, the deleted rows will be returned.

The only limitation of the ``RETURNING`` clause is that it can only consist of
columns from tables listed in the query's ``FROM`` clause. To select all
columns from a particular table, you can simply pass in the :py:class:`Model`
class.

As another example, let's add a user and set their creation-date to the
server-generated current timestamp. We'll create and retrieve the new user's
ID, Email and the creation timestamp in a single query:

.. code-block:: python

    query = (User
             .insert(email='foo@bar.com', created=fn.now())
             .returning(User))  # Shorthand for all columns on User.

    # When using RETURNING, execute() returns a cursor.
    cursor = query.execute()

    # Get the user object we just inserted and log the data:
    user = cursor[0]
    logger.info('Created user %s (id=%s) at %s', user.email, user.id, user.created)

By default the cursor will return :py:class:`Model` instances, but you can
specify a different row type:

.. code-block:: python

    data = [{'name': 'charlie'}, {'name': 'huey'}, {'name': 'mickey'}]
    query = (User
             .insert_many(data)
             .returning(User.id, User.username)
             .dicts())

    for new_user in query.execute():
        print('Added user "%s", id=%s' % (new_user['username'], new_user['id']))

Just as with :py:class:`Select` queries, you can specify various :ref:`result row types <rowtypes>`.

.. _cte:

共通のテーブル表現
------------------------

Peewee supports the inclusion of common table expressions (CTEs) in all types
of queries. CTEs may be useful for:

* Factoring out a common subquery.
* Grouping or filtering by a column derived in the CTE's result set.
* Writing recursive queries.

To declare a :py:class:`Select` query for use as a CTE, use
:py:meth:`~SelectQuery.cte` method, which wraps the query in a :py:class:`CTE`
object. To indicate that a :py:class:`CTE` should be included as part of a
query, use the :py:meth:`Query.with_cte` method, passing a list of CTE objects.

単純な例
^^^^^^^^^^^^^^

For an example, let's say we have some data points that consist of a key and a
floating-point value. Let's define our model and populate some test data:

.. code-block:: python

    class Sample(Model):
        key = TextField()
        value = FloatField()

    data = (
        ('a', (1.25, 1.5, 1.75)),
        ('b', (2.1, 2.3, 2.5, 2.7, 2.9)),
        ('c', (3.5, 3.5)))

    # Populate data.
    for key, values in data:
        Sample.insert_many([(key, value) for value in values],
                           fields=[Sample.key, Sample.value]).execute()

Let's use a CTE to calculate, for each distinct key, which values were
above-average for that key.

.. code-block:: python

    # First we'll declare the query that will be used as a CTE. This query
    # simply determines the average value for each key.
    cte = (Sample
           .select(Sample.key, fn.AVG(Sample.value).alias('avg_value'))
           .group_by(Sample.key)
           .cte('key_avgs', columns=('key', 'avg_value')))

    # Now we'll query the sample table, using our CTE to find rows whose value
    # exceeds the average for the given key. We'll calculate how far above the
    # average the given sample's value is, as well.
    query = (Sample
             .select(Sample.key, Sample.value)
             .join(cte, on=(Sample.key == cte.c.key))
             .where(Sample.value > cte.c.avg_value)
             .order_by(Sample.value)
             .with_cte(cte))

We can iterate over the samples returned by the query to see which samples had
above-average values for their given group:

.. code-block:: pycon

    >>> for sample in query:
    ...     print(sample.key, sample.value)

    # 'a', 1.75
    # 'b', 2.7
    # 'b', 2.9

複雑な例
^^^^^^^^^^^^^^^

For a more complete example, let's consider the following query which uses
multiple CTEs to find per-product sales totals in only the top sales regions.
Our model looks like this:

.. code-block:: python

    class Order(Model):
        region = TextField()
        amount = FloatField()
        product = TextField()
        quantity = IntegerField()

Here is how the query might be written in SQL. This example can be found in
the `postgresql documentation <https://www.postgresql.org/docs/current/static/queries-with.html>`_.

.. code-block:: sql

    WITH regional_sales AS (
        SELECT region, SUM(amount) AS total_sales
        FROM orders
        GROUP BY region
      ), top_regions AS (
        SELECT region
        FROM regional_sales
        WHERE total_sales > (SELECT SUM(total_sales) / 10 FROM regional_sales)
      )
    SELECT region,
           product,
           SUM(quantity) AS product_units,
           SUM(amount) AS product_sales
    FROM orders
    WHERE region IN (SELECT region FROM top_regions)
    GROUP BY region, product;

With Peewee, we would write:

.. code-block:: python

    reg_sales = (Order
                 .select(Order.region,
                         fn.SUM(Order.amount).alias('total_sales'))
                 .group_by(Order.region)
                 .cte('regional_sales'))

    top_regions = (reg_sales
                   .select(reg_sales.c.region)
                   .where(reg_sales.c.total_sales > (
                       reg_sales.select(fn.SUM(reg_sales.c.total_sales) / 10)))
                   .cte('top_regions'))

    query = (Order
             .select(Order.region,
                     Order.product,
                     fn.SUM(Order.quantity).alias('product_units'),
                     fn.SUM(Order.amount).alias('product_sales'))
             .where(Order.region.in_(top_regions.select(top_regions.c.region)))
             .group_by(Order.region, Order.product)
             .with_cte(regional_sales, top_regions))

再帰的 CTE
^^^^^^^^^^^^^^

Peewee supports recursive CTEs. Recursive CTEs can be useful when, for example,
you have a tree data-structure represented by a parent-link foreign key.
Suppose, for example, that we have a hierarchy of categories for an online
bookstore. We wish to generate a table showing all categories and their
absolute depths, along with the path from the root to the category.

We'll assume the following model definition, in which each category has a
foreign-key to its immediate parent category:

.. code-block:: python

    class Category(Model):
        name = TextField()
        parent = ForeignKeyField('self', backref='children', null=True)

To list all categories along with their depth and parents, we can use a
recursive CTE:

.. code-block:: python

    # Define the base case of our recursive CTE. This will be categories that
    # have a null parent foreign-key.
    Base = Category.alias()
    level = Value(1).alias('level')
    path = Base.name.alias('path')
    base_case = (Base
                 .select(Base.name, Base.parent, level, path)
                 .where(Base.parent.is_null())
                 .cte('base', recursive=True))

    # Define the recursive terms.
    RTerm = Category.alias()
    rlevel = (base_case.c.level + 1).alias('level')
    rpath = base_case.c.path.concat('->').concat(RTerm.name).alias('path')
    recursive = (RTerm
                 .select(RTerm.name, RTerm.parent, rlevel, rpath)
                 .join(base_case, on=(RTerm.parent == base_case.c.id)))

    # The recursive CTE is created by taking the base case and UNION ALL with
    # the recursive term.
    cte = base_case.union_all(recursive)

    # We will now query from the CTE to get the categories, their levels,  and
    # their paths.
    query = (cte
             .select_from(cte.c.name, cte.c.level, cte.c.path)
             .order_by(cte.c.path))

    # We can now iterate over a list of all categories and print their names,
    # absolute levels, and path from root -> category.
    for category in query:
        print(category.name, category.level, category.path)

    # Example output:
    # root, 1, root
    # p1, 2, root->p1
    # c1-1, 3, root->p1->c1-1
    # c1-2, 3, root->p1->c1-2
    # p2, 2, root->p2
    # c2-1, 3, root->p2->c2-1

外部キーと JOIN
----------------------

This section have been moved into its own document: :ref:`relationships`.
