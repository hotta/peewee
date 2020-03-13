.. _query-builder:

クエリービルダー
===========================

Peewee の高レベルの :py:class:`Model` と :py:class:`Field` API は,
低レベルの :py:class:`Table` と :py:class:`Column` という相手役の元に構築されています.
これら低レベル API はその相手役である高レベル API ほど詳しくはドキュメント化されていないのですが,
このドキュメントではその概要を利用例とともにお贈りすることで, あなたの実験の手助けになれればと思っています.

説明のために, 以下のスキーマを使います:

.. code-block:: sql

    CREATE TABLE "person" (
        "id" INTEGER NOT NULL PRIMARY KEY,
        "first" TEXT NOT NULL,
        "last" TEXT NOT NULL);

    CREATE TABLE "note" (
        "id" INTEGER NOT NULL PRIMARY KEY,
        "person_id" INTEGER NOT NULL,
        "content" TEXT NOT NULL,
        "timestamp" DATETIME NOT NULL,
        FOREIGN KEY ("person_id") REFERENCES "person" ("id"));

    CREATE TABLE "reminder" (
        "id" INTEGER NOT NULL PRIMARY KEY,
        "note_id" INTEGER NOT NULL,
        "alarm" DATETIME NOT NULL,
        FOREIGN KEY ("note_id") REFERENCES "note" ("id"));

テーブルを定義する
---------------------

これらのテーブルを扱うにあたり, :py:class:`Table` オブジェクトを定義するやり方に二種類あります:

.. code-block:: python

    # 明示的にカラムを宣言する
    Person = Table('person', ('id', 'first', 'last'))

    Note = Table('note', ('id', 'person_id', 'content', 'timestamp'))

    # カラムを宣言しない. それらは ".c" マジック属性を使ってアクセスできる.
    Reminder = Table('reminder')

基本的にはデータベースに対して私達のテーブルを :py:meth:`~Table.bind` します.これにより,
そのテーブルに対するクエリーを実行するたびに毎回 明示的にデータベースを渡す手間を省きます:


.. code-block:: python

    db = SqliteDatabase('my_app.db')
    Person = Person.bind(db)
    Note = Note.bind(db)
    Reminder = Reminder.bind(db)

Select クエリー
------------------

note の先頭３つを select してその中身を表示するには以下のように書きます:

.. code-block:: python

    query = Note.select().order_by(Note.timestamp).limit(3)
    for note_dict in query:
        print(note_dict['content'])

.. note::
    デフォルトでは行は辞書として返されます. 必要であれば行データに対して
    :py:meth:`~BaseQuery.tuples` , :py:meth:`~BaseQuery.namedtuples` , :py:meth:`~BaseQuery.objects` 
    メソッドにより別のコンテナを指定します.

ここでは何もカラムを指定していないので, note の :py:class:`Table` 
コンストラクタで定義されたすべてのカラムが select されます.
reminder テーブルについてはカラムを一切指定していないので,このやり方は動きません.

2018 年に発行されたすべての note をその作成者とともに select するには :py:meth:`~BaseQuery.join`
を使います. さらに行が *名前付きタプル* オブジェクトとして返されるように要求します:

.. code-block:: python

    query = (Note
             .select(Note.content, Note.timestamp, Person.first, Person.last)
             .join(Person, on=(Note.person_id == Person.id))
             .where(Note.timestamp >= datetime.date(2018, 1, 1))
             .order_by(Note.timestamp)
             .namedtuples())

    for row in query:
        print(row.timestamp, '-', row.content, '-', row.first, row.last)

誰が最も頻繁に情報を発信していたかを問い合わせてみましょう. すなわち,
最も多くのメモ(note)を作成した人たちを取得します. これは SQL 関数(COUNT)の利用例となりますが,
``fn`` オブジェクトを使って実現できます.

.. code-block:: python

    name = Person.first.concat(' ').concat(Person.last)
    query = (Person
             .select(name.alias('name'), fn.COUNT(Note.id).alias('count'))
             .join(Note, JOIN.LEFT_OUTER, on=(Note.person_id == Person.id))
             .group_by(name)
             .order_by(fn.COUNT(Note.id).desc()))
    for row in query:
        print(row['name'], row['count'])

There are a couple things to note in the above query:
上記のクエリーの中で, いくつか重要なことがあります:

* まず (``name``) 変数の中に評価式を入れ, それをクエリーで使います.
* SQL 関数は ``fn.<function>(...)`` で呼び出します. 引数はあたかも通常の Python の関数のようにして渡します.
* カラムもしくは計算で使う名前を指定するために :py:meth:`~ColumnBase.alias` メソッドが使われます.

より複雑な例として, すべての人とその content, およびそれぞれの人が最後に発行した note の]
timestamp のリストを生成してみましょう.これを行うには, 結局同じクエリーの中で note
テーブルを２回, それぞれ異なったコンテキストで使う必要があります. このために, 
テーブルの別名(alias)を使う必要があります.


.. code-block:: python

    # それぞれの人について, 最も最近の note のタイムスタンプを計算するクエリーから始める.
    NA = Note.alias('na')
    max_note = (NA
                .select(NA.person_id, fn.MAX(NA.timestamp).alias('max_ts'))
                .group_by(NA.person_id)
                .alias('max_note'))

    # つぎに note テーブルから select し, サブクエリーと person テーブルの
    # 両方を JOIN して 結果セットを構築する.
    query = (Note
             .select(Note.content, Note.timestamp, Person.first, Person.last)
             .join(max_note, on=((max_note.c.person_id == Note.person_id) &
                                 (max_note.c.max_ts == Note.timestamp)))
             .join(Person, on=(Note.person_id == Person.id))
             .order_by(Person.first, Person.last))

    for row in query.namedtuples():
        print(row.first, row.last, ':', row.timestamp, '-', row.content)

JOIN の中では *max_note* サブクエリーへの JOIN を記載し, ".c" 
マジック属性を使ってサブクエリーの中のカラムを参照できます. これにより *max_note.c.max_ts*
は "max_note サブクエリー由来の max_tx カラム" に変換されます.

".c" マジック属性を使うと, reminder テーブルのように明示的にカラムを定義していなくても,
テーブルのカラムへのアクセスが可能になります. 以下は本日の reminder を,
それと関連する note の content とともに取得するシンプルなクエリーです:


.. code-block:: python

    today = datetime.date.today()
    tomorrow = today + datetime.timedelta(days=1)

    query = (Reminder
             .select(Reminder.c.alarm, Note.content)
             .join(Note, on=(Reminder.c.note_id == Note.id))
             .where(Reminder.c.alarm.between(today, tomorrow))
             .order_by(Reminder.c.alarm))
    for row in query:
        print(row['alarm'], row['content'])

.. note::
    混乱を避けるため, カラムを明示的に定義したテーブルに対しては,
    ".c" 属性は動かないようになっています.

Insert クエリー
-------------------

データの insert は単純です. :py:meth:`~Table.insert` に対してデータを指定するのには,
二通りの方法があります(いずれの方法についても, 新しい行の ID が返されます):

.. code-block:: python

    # キーワード引数を使う:
    zaizee_id = Person.insert(first='zaizee', last='cat').execute()

    # カラムと値のマッピングを使う:
    Note.insert({
        Note.person_id: zaizee_id,
        Note.content: 'meeeeowwww',
        Note.timestamp: datetime.datetime.now()}).execute()

データの一括 insert は簡単です. 以下のいずれかを渡してあげるだけです:

* 辞書のリスト(すべて同じキー/カラムでなければなりません)
* タプルのリスト(カラムが明示的に指定されている場合)

使用例:

.. code-block:: python

    people = [
        {'first': 'Bob', 'last': 'Foo'},
        {'first': 'Herb', 'last': 'Bar'},
        {'first': 'Nuggie', 'last': 'Bar'}]

    # 複数の行を insert すると, 最後に insert された行の ID が返される.
    last_id = Person.insert(people).execute()

    # 行タプルを指定することも可能(どのカラムとタプル値が対応するのかを Peewee に教えてやる必要あり).
    people = [
        ('Bob', 'Foo'),
        ('Herb', 'Bar'),
        ('Nuggie', 'Bar')]
    Person.insert(people, columns=[Person.first, Person.last]).execute()

Update クエリー
-----------------------

:py:meth:`~Table.update` クエリーは, キーワード引数とカラムから値へのマッピング辞書のいずれも受け付けます.
:py:meth:`~Table.insert` と同様です.


使用例:

.. code-block:: python

    # "Bob" は自分の姓を "Foo" から "Baze" に変更した.
    nrows = (Person
             .update(last='Baze')
             .where((Person.first == 'Bob') &
                    (Person.last == 'Foo'))
             .execute())

    # カラム→値のマッピング辞書を使う
    nrows = (Person
             .update({Person.last: 'Baze'})
             .where((Person.first == 'Bob') &
                    (Person.last == 'Foo'))
             .execute())

表現式を値として使うことでアトミックな update を行うこともできます. *PageView* テーブルのある
URL について, ページビューのカウントをアトミックにインクリメントする必要がある場合の例を示します:

.. code-block:: python

    # アトミックな update を行う:
    (PageView
     .update({PageView.count: PageView.count + 1})
     .where(PageView.url == some_url)
     .execute())

Delete クエリー
---------------------

:py:meth:`~Table.delete` は何も引数を取らないので, もっともシンブルです:

.. code-block:: python

    # 2018 年以前に作成された note をすべて削除して, 削除件数を返す.
    n = Note.delete().where(Note.timestamp < datetime.date(2018, 1, 1)).execute()

DELETE(と UPDATE)クエリーは JOIN をサポートしません.
関連するテーブルの値に基づいて行を削除するにはサブクエリーを使います.
たとえば, 以下は姓が "Foo" であるユーザの note をすべて削除する例です:


.. code-block:: python

    # 姓が "Foo" であるユーザの ID を取得.
    foo_people = Person.select(Person.id).where(Person.last == 'Foo')

    # 直前のクエリーに ID がある人による note をすべて削除.
    Note.delete().where(Note.person_id.in_(foo_people)).execute()

クエリーオプション
---------------------

Peewee 2.x で提供されている抽象化に関する基本的な制限のひとつに,
指定されたモデルクラスへのリレーションがないような構造化クエリーを表現するためのクラスがなかったというのがあります.


この例では, サブクエリーに対する集約値を計算することがあるかもしれません。
たとえば :py:meth:`~SelectBase.count` メソッドは人氏のクエリーの行数を返しますが,
これはクエリーをラップすることにより実装されています:


.. code-block:: sql

    SELECT COUNT(1) FROM (...)

これを Peewee で実現するには, 以下のような方法で実装することになります:

.. code-block:: python

    def count(query):
        # Select([source1, ... sourcen], [column1, ...columnn])
        wrapped = Select(from_list=[query], columns=[fn.COUNT(SQL('1'))])
        curs = wrapped.tuples().execute(db)
        return curs[0][0]       # 結果の１行目の最初のカラムを返す

実際には :py:meth:`~SelectBase.scalar` メソッドを使うと, もっと簡潔に表現できます.
これは集約クエリーからの値を返すのに適しています:

.. code-block:: python

    def count(query):
        wrapped = Select(from_list=[query], columns=[fn.COUNT(SQL('1'))])
        return wrapped.scalar(db)

:ref:`query_examples` ドキュメントにはより複雑な例があります. 
これは, 予約対象の時間枠が最も空いている施設を見つけるようなクエリーです:

The SQL we wish to express is:

.. code-block:: sql

    SELECT facid, total FROM (
      SELECT facid, SUM(slots) AS total,
             rank() OVER (order by SUM(slots) DESC) AS rank
      FROM bookings
      GROUP BY facid
    ) AS ranked
    WHERE rank = 1

これを, 外側のクエリーで素の :py:class:`Select` を使い, かなりすっきりと表現できます:

.. code-block:: python

    # 読みやすいように, 順位の表現を変数に保存しておく.
    rank_expr = fn.rank().over(order_by=[fn.SUM(Booking.slots).desc()])

    subq = (Booking
            .select(Booking.facility, fn.SUM(Booking.slots).alias('total'),
                    rank_expr.alias('rank'))
            .group_by(Booking.facility))

    # 素の "Select" を使って外側のクエリーを作成する
    query = (Select(columns=[subq.c.facid, subq.c.total])
             .from_(subq)
             .where(subq.c.rank == 1)
             .tuples())

    # 結果をイテレートして施設 ID と合計を取り出す
    for facid, total in query.execute(db):
        print(facid, total)

別の例として, 再帰の共有テーブル表現を使ってフィボナッチ数列の最初の 10 個を計算してみましょう.

.. code-block:: python

    base = Select(columns=(
        Value(1).alias('n'),
        Value(0).alias('fib_n'),
        Value(1).alias('next_fib_n'))).cte('fibonacci', recursive=True)

    n = (base.c.n + 1).alias('n')
    recursive_term = Select(columns=(
        n,
        base.c.next_fib_n,
        base.c.fib_n + base.c.next_fib_n)).from_(base).where(n < 10)

    fibonacci = base.union_all(recursive_term)
    query = fibonacci.select_from(fibonacci.c.n, fibonacci.c.fib_n)

    results = list(query.execute(db))

    # 以下のリストが生成されます:
    [{'fib_n': 0, 'n': 1},
     {'fib_n': 1, 'n': 2},
     {'fib_n': 1, 'n': 3},
     {'fib_n': 2, 'n': 4},
     {'fib_n': 3, 'n': 5},
     {'fib_n': 5, 'n': 6},
     {'fib_n': 8, 'n': 7},
     {'fib_n': 13, 'n': 8},
     {'fib_n': 21, 'n': 9},
     {'fib_n': 34, 'n': 10}]

さらに
-------------

SQL AST を記述するのに使われるさまざまなクラスの解説については,
:ref:`クエリービルダーの API documentation <query-builder-api>` を参照してください.

より深く学びたい場合, `Peewee プロジェクトのソースコード <https://github.com/coleifer/peewee>`_
もチェックしてみてください.
