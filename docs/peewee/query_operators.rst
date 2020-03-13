.. _query-operators:

クエリー演算子
========================

peeweeでは以下の形式の比較方法をサポートしています:

================ =======================================
比較              意味
================ =======================================
``==``           x と y が等しい
``<``            x は y 未満
``<=``           x は y 以下
``>``            x は y より大きい
``>=``           x は y 以上
``!=``           x は y と異なる
``<<``           x IN y, y はリストまたはクエリー
``>>``           x IS y, y は None または NULL
``%``            x LIKE y, y はワイルドカードを含んでもよい
``**``           x ILIKE y, y はワイルドカードを含んでもよい
``^``            x XOR y
``~``            否定の単項演算子 (すなわち NOT x)
================ =======================================

すでにオーバーライド可能な演算子記号を使い尽くしてしまっているため,
以下のクエリー演算子をメソッドとして用意しています:

======================= ===============================================
メソッド                 意味
======================= ===============================================
``.in_(value)``         IN 探索 (``<<`` と同じ).
``.not_in(value)``      NOT IN 探索.
``.is_null(is_null)``   IS NULL または IS NOT NULL. ブール型パラメータも.
``.contains(substr)``   サブ文字列のワイルドカード検索.
``.startswith(prefix)`` ``prefix`` で始まる値の検索.
``.endswith(suffix)``   ``suffix`` で終わる値の検索.
``.between(low, high)`` ``low`` と ``high`` の間にある値の検索.
``.regexp(exp)``        正規表現マッチング（大文字小文字を区別する）.
``.iregexp(exp)``       正規表現マッチング（大文字小文字を区別しない）.
``.bin_and(value)``     バイナリの AND.
``.bin_or(value)``      バイナリの OR.
``.concat(other)``      ``||`` で２つの文字列またはオブジェクトを連結.
``.distinct()``         そのカラムを DISTINCT select としてマーク.
``.collate(collation)`` そのカラムに対して指定した照合順序を適用.
``.cast(type)``         そのカラムの値を指定した型でキャスト.
======================= ===============================================

論理演算を使って句を連結する際に使う演算子:

================ ==================== ======================================================
演算子            意味                 使用例
================ ==================== ======================================================
``&``            AND                  ``(User.is_active == True) & (User.is_admin == True)``
``|`` (pipe)     OR                   ``(User.is_admin) | (User.is_superuser)``
``~``            NOT (否定単項演算子)  ``~(User.username.contains('admin'))``
================ ==================== ======================================================

これらのクエリー演算子の使用例:

.. code-block:: python

    # username が "charlie" であるユーザを見つける.
    User.select().where(User.username == 'charlie')

    # username が [charlie, huey, mickey] のいずれかであるユーザを見つける.
    User.select().where(User.username.in_(['charlie', 'huey', 'mickey']))

    Employee.select().where(Employee.salary.between(50000, 60000))

    Employee.select().where(Employee.name.startswith('C'))

    Blog.select().where(Blog.title.contains(search_string))

次は式を結合する方法です. 比較はどんなに複雑でも構いません.

.. note::
  実際の比較部分は括弧で括られます. Python の演算子の優先度の都合により,
  比較部分は括弧で括られている必要があります.

.. code-block:: python

    # アクティブな管理者であるユーザを見つける.
    User.select().where(
      (User.is_admin == True) &
      (User.is_active == True))

    # 管理者もしくはスーパーユーザであるユーザを見つける.
    User.select().where(
      (User.is_admin == True) |
      (User.is_superuser == True))

    # 管理者でない(NOT IN)ユーザのツイートを見つける.
    admins = User.select().where(User.is_admin == True)
    non_admin_tweets = Tweet.select().where(Tweet.user.not_in(admins))

    # 自分のfriendsでない（知らない）ユーザを見つける
    friends = User.select().where(User.username.in_(['charlie', 'huey', 'mickey']))
    strangers = User.select().where(User.id.not_in(friends))

.. warning::
    自分のクエリーの中で Python の``in``, ``and``, ``or``, ``not`` といった
    演算子を使いたくなる誘惑に駆られるかもしれませんが,これらは **動きません.**
    ``in`` 式の戻り値は,常に強制的にブール値に変更されます.同様に,``and``, ``or``, 
    ``not`` の引数もブール値になってしまい,これをオーバーオードすることはできません.

    このため,以下のことだけを覚えておきましょう:

    * ``in`` と ``not in`` の代わりに ``.in_()`` と ``.not_in()`` を使う
    * ``and`` の代わりに ``&`` を使う
    *  ``or`` の代わりに ``|`` を使う
    *  ``not`` の代わりに ``~`` を使う
    * ``is None`` や ``== None`` の代わりに  ``.is_null()`` を使う.
    * **論理演算子を使う場合は,比較部分全体を括弧で囲むのを忘れないこと.**

これ以外の例については, :ref:`expressions` の章を参照してください.

.. note::
  **SQLite における LIKE とILIKE**

  SQLite の ``LIKE`` はデフォルトではケース（大文字小文字）を区別しないため,
  peewee はケースを区別する検索の場合, SQLite の ``GLOB`` 機能を使います.
  glob 機能ではワイルドカードを表す際,通常のパーセント記号ではなくアスタリスク
  を使います. もし SQLite を使ってケースを区別する部分一致検索を行う場合,
  ワイルドカードにはアスタリスクを使うことを覚えておいてください.

Three valued logic
------------------

SQL の ``NULL`` を扱うために, 特別な書き方をする表現式があります:

* ``IS NULL``
* ``IS NOT NULL``
* ``IN``
* ``NOT IN``

``IS NULL`` や ``IN`` 演算子を否定演算子 (``~``) と組み合わせて使うことは
できますが,文脈をはっきりさせるため,明示的に ``IS NOT NULL`` と ``NOT IN``
を使う必要がある場合があります.

``IS NULL`` と ``IN`` を使うための最も簡単な方法は,演算子オーバーロード
を使う方法です:

.. code-block:: python

    # last login が NULL である User オブジェクトをすべて取得する
    User.select().where(User.last_login >> None)

    # username が指定されたリストに含まれるユーザを取得する
    usernames = ['charlie', 'huey', 'mickey']
    User.select().where(User.username << usernames)

演算子オーバーロードを使いたくない場合,代わりにフィールドメソッドを呼ぶこともできます:

.. code-block:: python

    # last login が NULL である User オブジェクトをすべて取得する
    User.select().where(User.last_login.is_null(True))

    # username が指定されたリストに含まれるユーザを取得する
    usernames = ['charlie', 'huey', 'mickey']
    User.select().where(User.username.in_(usernames))

前述とは反対の意味のクエリーを使いたい場合,単項否定演算子も使えます.ただし
文脈上正確に意図を示したい場合,特別な ``IS NOT`` / ``NOT IN`` 演算子が使えます:

.. code-block:: python

    # last login が NULL *でない* User オブジェクトをすべて取得する
    User.select().where(User.last_login.is_null(False))

    # これを単項否定演算子を使って行う.
    User.select().where(~(User.last_login >> None))

    # username が指定されたリストに *含まれない* ユーザを取得する
    usernames = ['charlie', 'huey', 'mickey']
    User.select().where(User.username.not_in(usernames))

    # これを単項否定演算子を使って行う.
    usernames = ['charlie', 'huey', 'mickey']
    User.select().where(~(User.username << usernames))

.. _custom-operators:

ユーザ定義演算子を追加する
-----------------------------

オーバーロードできそうな Python の演算子を使い尽くしてしまったため,peewee
にはたとえば ``剰余`` のように、存在しない演算子があります.前述の表の中に
自分が使いたい演算子がなかった場合,簡単に自分専用の演算子を追加できます.

SQlite で ``剰余`` サポートを追加するには以下のようにします:

.. code-block:: python

    from peewee import *
    from peewee import Expression # 式のビルド部分

    def mod(lhs, rhs):
        return Expression(lhs, '%', rhs)

これらのカスタム演算子を使うことで、より直感的なクエリーをビルド(構築)
できるようになりました:

.. code-block:: python

    # id が奇数のユーザ
    User.select().where(mod(User.id, 2) == 0)

これ以外の使用例については ``playhouse.postgresql_ext`` モジュールの
ソースを参照してみてください.この中には postgresql の hstore に固有の
演算子が多数含まれています.

.. _expressions:

表現式(Expressions)
-------------------

Peewee はシンプルで表現力豊かな,かつpython的な方法によるSQLクエリー構築が
できるようにデザインされています.この章では一般的な表現式についての概観を
みてみましょう.

Peewee では２つのタイプのオブジェクトを組み合わせて表現式を生成します:

* :py:class:`Field` インスタンス
* SQL の集約関数と :py:class:`fn` を使った関数

ユーザ名その他の項目を持つ,シンプルな "User" モデルを考えてみましょう.
それは以下のようになります:

.. code-block:: python

    class User(Model):
        username = CharField()
        is_admin = BooleanField()
        is_active = BooleanField()
        last_login = DateTimeField()
        login_count = IntegerField()
        failed_logins = IntegerField()

比較の際は :ref:`query-operators` を使います:

.. code-block:: python

    # username が 'charlie' と等しい
    User.username == 'charlie'

    # ログイン回数が５回未満のユーザ
    User.login_count < 5

比較の際は **ビットごとの** **and** と **or** を組み合わせます.
演算子の優先順位は python によって制御され,比較は入れ子にして
任意の深さにできます:

.. code-block:: python

    # admin でかつ今日ログインしたユーザ
    (User.is_admin == True) & (User.last_login >= today)

    # ユーザ名が charlie または charles
    (User.username == 'charlie') | (User.username == 'charles')

比較は関数を使って行うこともできます:

.. code-block:: python

    # ユーザ名が 'g' または 'G' で始まるユーザ
    fn.Lower(fn.Substr(User.username, 1, 1)) == 'g'

表現式を別の表現式と比較することができるので,ちょっとおもしろいことをやってみましょう.
表現式では算術演算子もサポートしています:

.. code-block:: python

    # ログイン失敗の回数が成功回数の半分を超えるものの,
    # 少なくとも10回はログインしたことがあるユーザ
    (User.failed_logins > (User.login_count * .5)) & (User.login_count > 10)

表現式では *アトミック(途中の割り込みがないことが保証されている)な更新* が行えます:

.. code-block:: python

    # ユーザがログインした時,それらのログイン回数をインクリメントしたい:
    User.update(login_count=User.login_count + 1).where(User.id == user_id)

表現式はクエリー内のすべての部分で使えます.これは新機軸です!


行の値(row values)
^^^^^^^^^^^^^^^^^^

多くのデータベースでは `行の値(row values) <https://www.sqlite.org/rowvalue.html>`_
をサポートしていますが,これは python の `タプル(tuple)` オブジェクトとよく似ています.
Peewee では表現式における行の値は :py:class:`Tuple` 経由で利用可能です.
以下に例を示します.

.. code-block:: python

    # もしあなたのスキーマで何らかの理由で日付を ("year", "month", "day") という
    # 別々のカラムに格納している場合でも,行の値を使って指定月に属するすべての行を
    # 見つけることが可能です:
    Tuple(Event.year, Event.month) == (2019, 1)

行の値に関するより一般的な使い方を,単一表現式の中のサブクエリーから生成される
複数カラムの場合と比較してみましょう.これらのタイプのクエリーを表現するための
他の方法もありますが,行の値を利用する方法は完結で読みやすいアプローチです.

ここでは例として "EventLog" というテーブルがあり,この中にはイベントタイプと
イベントソース,その他のメタデータが入っているとします.さらに "IncidentLog"
もあり,これにはインシデントタイプ,インシデントソース,メタデータのカラムが
入っています.行の値を使えば,インシデントと特定のイベントを関連付けることが
可能です:

.. code-block:: python

    class EventLog(Model):
        event_type = TextField()
        source = TextField()
        data = TextField()
        timestamp = TimestampField()

    class IncidentLog(Model):
        incident_type = TextField()
        source = TextField()
        traceback = TextField()
        timestamp = TimestampField()

    # 本日発生したインシデントのタイプとソースの一覧をすべて取得する
    incidents = (IncidentLog
                 .select(IncidentLog.incident_type, IncidentLog.source)
                 .where(IncidentLog.timestamp >= datetime.date.today()))

    # 本日発生したイベントの中で,タイプとソースがインシデントに関連する
    # ものをすべて見つける.
    events = (EventLog
              .select()
              .where(Tuple(EventLog.event_type, EventLog.source).in_(incidents))
              .order_by(EventLog.timestamp))

このタイプのクエリーを表現するための他の方法としては, :ref:`join <relationships>`
や :ref:`サブクエリーに対する join <join-subquery>` が使えます.
上記の例は,単に :py:class:`Tuple` を使うためのアイデアを提供したに過ぎません.

新しいデータがサブクエリーから発生した場合,行の値を使ってテーブル内の複数のカラムを
更新することもできます.この利用例としては
`here <https://www.sqlite.org/rowvalue.html#update_multiple_columns_of_a_table_based_on_a_query>`_
を参照してください.

SQL 関数
-------------

``COUNT()`` や ``SUM()`` といった SQL 関数は, :py:func:`fn` ヘルパー: を
使って表現できます:

.. code-block:: python

    # すべてのユーザと,彼らがつぶやいた多数のツイートを取得する.
    # 結果はツイート数の多い方から降順にソートする.
    query = (User
             .select(User, fn.COUNT(Tweet.id).alias('tweet_count'))
             .join(Tweet, JOIN.LEFT_OUTER)
             .group_by(User)
             .order_by(fn.COUNT(Tweet.id).desc()))
    for user in query:
        print('%s -- %s tweets' % (user.username, user.tweet_count))

``fn`` ヘルパーは,任意の SQL 関数をあたかもメソッドであるかのように見せて
くれます.パラメータはフィールド,値,サブクエリーに加えて,さらにネストした
関数も指定できます。


ネストした関数呼び出し
^^^^^^^^^^^^^^^^^^^^^^

ここであなたは,ユーザ名が *a* で始まるユーザの一覧を取得する必要があるとします.
このやり方はいくつかありますが,*LOWER* や *SUBSTR* といった SQL 関数を使える
メソッドが1つあります.任意の SQL 関数を使いたい場合は, :py:func:`fn` という
特別なオブジェクトを使ってクエリーを構築します.

.. code-block:: python

    # ユーザの id とユーザ名, およびユーザ名の先頭1文字を小文字にした
    # ものを select する
    first_letter = fn.LOWER(fn.SUBSTR(User.username, 1, 1))
    query = User.select(User, first_letter.alias('first_letter'))

    # 別の方法として,ユーザ名が 'a' で始まるユーザだけを select することも可能
    a_users = User.select().where(first_letter == 'a')

    >>> for user in a_users:
    ...    print(user.username)

SQL ヘルパー
------------

単に任意の SQL 文を渡したいという場合もあると思います.この場合,特別な
:py:class:`SQL` クラスが使えます.ユースケースのひとつして,別名を参照
したい場合を示します:

.. code-block:: python

    # ユーザテーブルに問い合わせを行い,特定のユーザについてツイート数の
    # 注釈を付ける.
    query = (User
             .select(User, fn.Count(Tweet.id).alias('ct'))
             .join(Tweet)
             .group_by(User))

    # 次に "ct" という別名を付けられた回数フィールドでソートする.
    query = query.order_by(SQL('ct'))

    # これは当然以下のようにも書ける:
    query = query.order_by(fn.COUNT(Tweet.id))

Peewee で手作りの SQL ステートメントを実行したい場合,以下の２つの方法がある:

1. 任意のタイプのクエリーを実行するための :py:meth:`Database.execute_sql`
2. ``SELECT`` クエリーを発行してモデルのインスタンスを返す :py:class:`RawQuery`

セキュリティと SQL インジェクション
---------------------------------------

peewee はデフォルトでクエリーをパラメータ化するため,ユーザから渡されたパラメータは
すべてエスケープされます.このルールにおける唯一の例外は,生の SQL クエリーを書いたり,
信頼できないデータを含む恐れがある ``SQL`` オブジェクトを渡す場合です.危険性を軽減
するため,いかなるユーザ定義データもクエリーパラメータとして渡し,実際の SQL クエリー
の一部として渡すことがないようにしてください:

.. code-block:: python

    # やっちゃダメ！
    query = MyModel.raw('SELECT * FROM my_table WHERE data = %s' % (user_data,))

    # 大丈夫. `user_data` はクエリーへのパラメータとして扱われます.
    query = MyModel.raw('SELECT * FROM my_table WHERE data = %s', user_data)

    # やっちゃダメ！
    query = MyModel.select().where(SQL('Some SQL expression %s' % user_data))

    # 大丈夫. `user_data` はパラメータとして扱われます.
    query = MyModel.select().where(SQL('Some SQL expression %s', user_data))

.. note::
    MySQL と Postgresql は ``'%s'`` をパラメータの印として使用します.一方
    SQLite は ``'?'`` を使います.お使いのデータベースに適した文字を使うように
    心がけてください.このパラメータは :py:attr:`Database.param` をチェックする
    ことで見つけらることができます.
