.. _query_examples:

クエリーの例
=====================

以下のクエリーの例は `PostgreSQL Exercises <https://pgexercises.com/>`_
のサイトから拝借してきたものです.サンプルのデータセットは
`getting started page <https://pgexercises.com/gettingstarted.html>`_
にあります.

これらの例で使われているスキーマのビジュアル表現です:

.. image:: schema-horizontal.png

モデルの定義
-----------------

データを扱うには,この図にあるテーブルに対応するモデルクラスを定義してあげる必要があります.

.. note::
    特定のフィールドについて,明示的にカラム名を指定しているケースがあります.
    これにより, 私達のモデルは postgres Exercises で使われるデータベーススキーマとの互換性があります.

.. code-block:: python

    from functools import partial
    from peewee import *


    db = PostgresqlDatabase('peewee_test')

    class BaseModel(Model):
        class Meta:
            database = db

    class Member(BaseModel):
        memid = AutoField()     # 自動インクリメントのプライマリキー
        surname = CharField()
        firstname = CharField()
        address = CharField(max_length=300)
        zipcode = IntegerField()
        telephone = CharField()
        recommendedby = ForeignKeyField('self', backref='recommended',
                                        column_name='recommendedby', null=True)
        joindate = DateTimeField()

        class Meta:
            table_name = 'members'


    # 便宜上,通貨を格納するのに適した decimal のフィールドを宣言しています.
    MoneyField = partial(DecimalField, decimal_places=2)


    class Facility(BaseModel):
        facid = AutoField()
        name = CharField()
        membercost = MoneyField()
        guestcost = MoneyField()
        initialoutlay = MoneyField()
        monthlymaintenance = MoneyField()

        class Meta:
            table_name = 'facilities'


    class Booking(BaseModel):
        bookid = AutoField()
        facility = ForeignKeyField(Facility, column_name='facid')
        member = ForeignKeyField(Member, column_name='memid')
        starttime = DateTimeField()
        slots = IntegerField()

        class Meta:
            table_name = 'bookings'


スキーマの生成
---------------

PostgreSQL Exercises のサイトから SQL ファイルをダウンロードしてきた場合は,
以下のコマンドを使って PostgreSQL データベースに対してデータをロードできます::

    createdb peewee_test
    psql -U postgres -f clubdata.sql -d peewee_test -x -q

Peewee でサンプルデータのロードを行わずにスキーマを生成する場合は以下の
コマンドを使ってください:

.. code-block:: python

    # すでにデータベース "peewee_test" があるものとします.
    db.create_tables([Member, Facility, Booking])


基本練習
---------------

このカテゴリーでは SQL の基礎について学びます.ここでカバーするのは select
と where 句, case 表現, union その他いくつかのこまごましたものです.

すべてを取り出す
^^^^^^^^^^^^^^^^^^^

facilities（施設）テーブルからすべての情報を取り出します.

.. code-block:: sql

    SELECT * FROM facilities

.. code-block:: python

    # デフォルトでは, select() に対してフィールドが明示的に渡されない場合,
    # すべてのフィールドが取り出されます.
    query = Facility.select()

テーブルから特定のカラムを取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設名とメンバーの料金を取り出します.

.. code-block:: sql

    SELECT name, membercost FROM facilities;

.. code-block:: python

    query = Facility.select(Facility.name, Facility.membercost)

    # 繰り返し(イテレータ)で取り出す場合:
    for facility in query:
        print(facility.name)

取り出すべき行を制御する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設からメンバー料金があるもののリストを取り出します.

.. code-block:: sql

    SELECT * FROM facilities WHERE membercost > 0

.. code-block:: python

    query = Facility.select().where(Facility.membercost > 0)

取り出すべき行をコントロールする - part 2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設の中でメンバーの料金があるものについて,その料金がひと月の維持費の50分の1
より小さなものに限ったリストを取り出します.
id, name, cost, monthlymaintenance が返されます.

.. code-block:: sql

    SELECT facid, name, membercost, monthlymaintenance
    FROM facilities
    WHERE membercost > 0 AND membercost < (monthlymaintenance / 50)

.. code-block:: python

    query = (Facility
             .select(Facility.facid, Facility.name, Facility.membercost,
                     Facility.monthlymaintenance)
             .where(
                 (Facility.membercost > 0) &
                 (Facility.membercost < (Facility.monthlymaintenance / 50))))

基本的な文字列検索
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

名前の中に 'Tennis' という単語を含む施設の一覧を取得するにはどうすれば
よいでしょうか？

.. code-block:: sql

    SELECT * FROM facilities WHERE name ILIKE '%tennis%';

.. code-block:: python

    query = Facility.select().where(Facility.name.contains('tennis'))

    # または指数演算子を使う. ワイルドカードを含む必要があることに注意:
    query = Facility.select().where(Facility.name ** '%tennis%')


複数の候補値に対するマッチング
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設のうちの ID が 1 または 5 のものの詳細を取り出すにはどうすればよいでしょう？
OR 演算子を使わずにやってみてください.

.. code-block:: sql

    SELECT * FROM facilities WHERE facid IN (1, 5);

.. code-block:: python

    query = Facility.select().where(Facility.facid.in_([1, 5]))

    # または:
    query = Facility.select().where((Facility.facid == 1) |
                                    (Facility.facid == 5))

結果を分類してバケツに入れる
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設について,その維持費の月額が $100 を超えるかどうかで 'cheap' または
'expensive' ラベルが付いたリストを取得するにはどうすればよいでしょうか？
それぞれの施設の名前と,その維持の状況を返します.

.. code-block:: sql

    SELECT name,
    CASE WHEN monthlymaintenance > 100 THEN 'expensive' ELSE 'cheap' END
    FROM facilities;

.. code-block:: python

    cost = Case(None, [(Facility.monthlymaintenance > 100, 'expensive')], 'cheap')
    query = Facility.select(Facility.name, cost.alias('cost'))

.. note:: 詳細は :py:class:`Case` のドキュメントを参照してください.


日付の操作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012年9月以降に加入したメンバーのリストを取得するにはどうすればよいでしょうか？
それぞれのメンバーの memid, surname, firstname, および加入日を返します.

.. code-block:: sql

    SELECT memid, surname, firstname, joindate FROM members
    WHERE joindate >= '2012-09-01';

.. code-block:: python

    query = (Member
             .select(Member.memid, Member.surname, Member.firstname, Member.joindate)
             .where(Member.joindate >= datetime.date(2012, 9, 1)))


重複を排除して結果をソートする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メンバーテーブルから先頭10個の名字を取り出して,それをソートしたリストを
作るにはどうしたらよいでしょうか？リストは重複してはならないものとします.

.. code-block:: sql

    SELECT DISTINCT surname FROM members ORDER BY surname LIMIT 10;

.. code-block:: python

    query = (Member
             .select(Member.surname)
             .order_by(Member.surname)
             .limit(10)
             .distinct())


複数のクエリーの結果を結合する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

何らかの理由により,すべての名字とすべての設備名をまとめたリストがほしいとします.

.. code-block:: sql

    SELECT surname FROM members UNION SELECT name FROM facilities;

.. code-block:: python

    lhs = Member.select(Member.surname)
    rhs = Facility.select(Facility.name)
    query = lhs | rhs

以下の演算子を使ってクエリーを組み合わせることができます:

* ``|`` - ``UNION``
* ``+`` - ``UNION ALL``
* ``&`` - ``INTERSECT``
* ``-`` - ``EXCEPT``

シンプルな集約
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最新のメンバーがサインアップした日付を知りたいとします.
この情報をどうやって持ってくればいいでしょうか？

.. code-block:: sql

    SELECT MAX(join_date) FROM members;

.. code-block:: python

    query = Member.select(fn.MAX(Member.joindate))
    # 手軽にスカラー値を取得したい場合は "scalar()" を使う:
    # max_join_date = query.scalar()

さらなる集約
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最も最近サインアップしたメンバーの加入日だけでなく,名前と名字も合わせて
取得したいとします.

.. code-block:: sql

    SELECT firstname, surname, joindate FROM members
    WHERE joindate = (SELECT MAX(joindate) FROM members);

.. code-block:: python

    # クエリーの中で同じテーブルを複数回参照する場合は "alias()" を使う.
    MemberAlias = Member.alias()
    subq = MemberAlias.select(fn.MAX(MemberAlias.joindate))
    query = (Member
             .select(Member.firstname, Member.surname, Member.joindate)
             .where(Member.joindate == subq))


JOIN とサブクエリー
--------------------

このカテゴリーでは,主にリレーショナルデータベースにおける基本概念である結合(join)について学びます.
結合を使うと,複数のテーブルの関連する情報をまとめて質問に答えることが可能です.
これは,クエリーを簡単にするために役に立つだけではありません:
join 機能がないとデータを非正規化せざるを得なくなり,複雑さが増すことでデータの内部的な整合性を保つことが困難になります.

このトピックでは内部,外部,自己結合に加えてサブクエリー(クエリーの中のクエリー)にも多少説明の時間を割くことにします.

メンバーの予約開始時刻を取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

'David Farrell' という名前のメンバーの予約開始時刻のリストを取得するには
どうすればよいでしょうか？

.. code-block:: sql

    SELECT starttime FROM bookings
    INNER JOIN members ON (bookings.memid = members.memid)
    WHERE surname = 'Farrell' AND firstname = 'David';

.. code-block:: python

    query = (Booking
             .select(Booking.starttime)
             .join(Member)
             .where((Member.surname == 'Farrell') &
                    (Member.firstname == 'David')))


テニスコートの予約開始時刻を使って練習
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

'2012-09-21' という日付でテニスコートの予約開始時刻のリストを取得するには
どうすればよいでしょうか？開始時刻と施設名のペアを,時刻でソートして返します.


.. code-block:: sql

    SELECT starttime, name
    FROM bookings
    INNER JOIN facilities ON (bookings.facid = facilities.facid)
    WHERE date_trunc('day', starttime) = '2012-09-21':: date
      AND name ILIKE 'tennis%'
    ORDER BY starttime, name;

.. code-block:: python

    query = (Booking
             .select(Booking.starttime, Facility.name)
             .join(Facility)
             .where(
                 (fn.date_trunc('day', Booking.starttime) == datetime.date(2012, 9, 21)) &
                 Facility.name.startswith('Tennis'))
             .order_by(Booking.starttime, Facility.name))

    # 結合された施設名をイテレータを使って取り出す:
    for booking in query:
        print(booking.starttime, booking.facility.name)


他のメンバーを推薦したことがあるメンバーを取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

他のメンバーを推薦したことがあるメンバーのリストを取り出すにはどうすればよいでしょうか？
リストには重複がないものとし,結果は(surname, firstname)でソートされているものとします.

.. code-block:: sql

    SELECT DISTINCT m.firstname, m.surname
    FROM members AS m2
    INNER JOIN members AS m ON (m.memid = m2.recommendedby)
    ORDER BY m.surname, m.firstname;

.. code-block:: python

    MA = Member.alias()
    query = (Member
             .select(Member.firstname, Member.surname)
             .join(MA, on=(MA.recommendedby == Member.memid))
             .order_by(Member.surname, Member.firstname))


すべてのメンバーのリストに合わせて,その推薦者も同時に取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メンバーの中で（もしあれば）その人の推薦者も合わせてリスト表示するには
どうすればよいでしょうか？結果は(surname,firstname)でソートされるものとします。

.. code-block:: sql

    SELECT m.firstname, m.surname, r.firstname, r.surname
    FROM members AS m
    LEFT OUTER JOIN members AS r ON (m.recommendedby = r.memid)
    ORDER BY m.surname, m.firstname

.. code-block:: python

    MA = Member.alias()
    query = (Member
             .select(Member.firstname, Member.surname, MA.firstname, MA.surname)
             .join(MA, JOIN.LEFT_OUTER, on=(Member.recommendedby == MA.memid))
             .order_by(Member.surname, Member.firstname))

    # To display the recommender's name when iterating:
    for m in query:
        print(m.firstname, m.surname)
        if m.recommendedby:
            print('  ', m.recommendedby.firstname, m.recommendedby.surname)


テニスコートを使ったことがあるメンバーを取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

テニスコートを使ったことがあるメンバーのリストを取り出すにはどうすればよいでしょうか？
出力にはコートの名前とメンバー名が結合して１カラムになったものを含みます.
またデータには重複がなく,またメンバー名でソートされているものとします.

.. code-block:: sql

    SELECT DISTINCT m.firstname || ' ' || m.surname AS member, f.name AS facility
    FROM members AS m
    INNER JOIN bookings AS b ON (m.memid = b.memid)
    INNER JOIN facilities AS f ON (b.facid = f.facid)
    WHERE f.name LIKE 'Tennis%'
    ORDER BY member, facility;

.. code-block:: python

    fullname = Member.firstname + ' ' + Member.surname
    query = (Member
             .select(fullname.alias('member'), Facility.name.alias('facility'))
             .join(Booking)
             .join(Facility)
             .where(Facility.name.startswith('Tennis'))
             .order_by(fullname, Facility.name)
             .distinct())


値段が高い予約のリストを取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012-09-14 の予約のうち,メンバー(もしくはゲスト)料金が $30 を超えるものの
リストを取り出すにはどうすればよいでしょうか？ゲストはメンバー(リストに現れる
料金は30分'時間枠'単位)とは別の料金体系を持っており,ゲストユーザーの ID は
常に 0 です.出力には施設名とメンバー名が結合して1カラムになったものと料金を
含みます.ソート順は利用料の降順であり,サブクエリーは使わないことにします.

.. code-block:: sql

    SELECT m.firstname || ' ' || m.surname AS member,
           f.name AS facility,
           (CASE WHEN m.memid = 0 THEN f.guestcost * b.slots
            ELSE f.membercost * b.slots END) AS cost
    FROM members AS m
    INNER JOIN bookings AS b ON (m.memid = b.memid)
    INNER JOIN facilities AS f ON (b.facid = f.facid)
    WHERE (date_trunc('day', b.starttime) = '2012-09-14') AND
     ((m.memid = 0 AND b.slots * f.guestcost > 30) OR
      (m.memid > 0 AND b.slots * f.membercost > 30))
    ORDER BY cost DESC;

.. code-block:: python

    cost = Case(Member.memid, (
        (0, Booking.slots * Facility.guestcost),
    ), (Booking.slots * Facility.membercost))
    fullname = Member.firstname + ' ' + Member.surname

    query = (Member
             .select(fullname.alias('member'), Facility.name.alias('facility'),
                     cost.alias('cost'))
             .join(Booking)
             .join(Facility)
             .where(
                 (fn.date_trunc('day', Booking.starttime) == datetime.date(2012, 9, 14)) &
                 (cost > 30))
             .order_by(SQL('cost').desc()))

    # 結果をイテレートする際は,名前付きタプルが最も扱いやすいでしょう:
    for row in query.namedtuples():
        print(row.member, row.facility, row.cost)


join を使ってすべてのメンバーのリストを推薦者と合わせて取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メンバーのうち(もしあれば)他の人を推薦したという情報を,JOINを使わずに取り出す
にはどうすればよいでしょうか？結果には重複がなく,それぞれの firstname + surname 
のペアが１カラムにまとめられ,かつそれを使ってソートされているものとします.

.. code-block:: sql

    SELECT DISTINCT m.firstname || ' ' || m.surname AS member,
       (SELECT r.firstname || ' ' || r.surname
        FROM cd.members AS r
        WHERE m.recommendedby = r.memid) AS recommended
    FROM members AS m ORDER BY member;

.. code-block:: python

    MA = Member.alias()
    subq = (MA
            .select(MA.firstname + ' ' + MA.surname)
            .where(Member.recommendedby == MA.memid))
    query = (Member
             .select(fullname.alias('member'), subq.alias('recommended'))
             .order_by(fullname))


サブクエリーを使って高価な予約のリストを取り出す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

"高価な予約のリストを取り出す" 練習では,若干面倒なロジックを抑制しています:
予約料金を計算は WHERE 句と CASE 文の両方で行う必要がありました.
これを単純化するために,サブクエリーを使って計算しています.


.. code-block:: sql

    SELECT member, facility, cost from (
      SELECT
      m.firstname || ' ' || m.surname as member,
      f.name as facility,
      CASE WHEN m.memid = 0 THEN b.slots * f.guestcost
      ELSE b.slots * f.membercost END AS cost
      FROM members AS m
      INNER JOIN bookings AS b ON m.memid = b.memid
      INNER JOIN facilities AS f ON b.facid = f.facid
      WHERE date_trunc('day', b.starttime) = '2012-09-14'
    ) as bookings
    WHERE cost > 30
    ORDER BY cost DESC;

.. code-block:: python

    cost = Case(Member.memid, (
        (0, Booking.slots * Facility.guestcost),
    ), (Booking.slots * Facility.membercost))

    iq = (Member
          .select(fullname.alias('member'), Facility.name.alias('facility'),
                  cost.alias('cost'))
          .join(Booking)
          .join(Facility)
          .where(fn.date_trunc('day', Booking.starttime) == datetime.date(2012, 9, 14)))

    query = (Member
             .select(iq.c.member, iq.c.facility, iq.c.cost)
             .from_(iq)
             .where(iq.c.cost > 30)
             .order_by(SQL('cost').desc()))

    # イテレートする際はdicts(辞書)を使ってみてください:
    for row in query.dicts():
        print(row['member'], row['facility'], row['cost'])


データの変更
--------------

データにクエリーをかけることはできるようになりましたが,次はおそらくデータベースに
データを書き込みたいと思うようになるはずです! このようにデータを変更する操作は,
まとめて Data Manipulation Language または DML と呼ばれています.

これまでの章では,実行したクエリーの結果を返していました.この章で述べるような
変更処理ではクエリーの結果を返しません.その代わり,意図に基づいて更新されたテーブル
の内容を表示します.

データをテーブルに insert する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このクラブでは新しい施設であるスパを増やそうとしています.私達はこれを facilities
テーブルに追加する必要があります.以下の値を使うことにします:
facid: 9, Name: 'Spa', membercost: 20, guestcost: 30, 
initialoutlay: 100000, monthlymaintenance: 800

.. code-block:: sql

    INSERT INTO "facilities" ("facid", "name", "membercost", "guestcost",
    "initialoutlay", "monthlymaintenance") VALUES (9, 'Spa', 20, 30, 100000, 800)

.. code-block:: python

    res = Facility.insert({
        Facility.facid: 9,
        Facility.name: 'Spa',
        Facility.membercost: 20,
        Facility.guestcost: 30,
        Facility.initialoutlay: 100000,
        Facility.monthlymaintenance: 800}).execute()

    # OR:
    res = (Facility
           .insert(facid=9, name='Spa', membercost=20, guestcost=30,
                   initialoutlay=100000, monthlymaintenance=800)
           .execute())


複数行データをテーブルに insert する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

直前の演習では施設の追加方法を学びました.次に複数の施設を1個のコマンドで
追加してみましょう.以下の値を使います:

facid: 9, Name: 'Spa', membercost: 20, guestcost: 30, initialoutlay: 100000,
monthlymaintenance: 800.

facid: 10, Name: 'Squash Court 2', membercost: 3.5, guestcost: 17.5,
initialoutlay: 5000, monthlymaintenance: 80.

.. code-block:: sql

    -- 前述 --

.. code-block:: python

    data = [
        {'facid': 9, 'name': 'Spa', 'membercost': 20, 'guestcost': 30,
         'initialoutlay': 100000, 'monthlymaintenance': 800},
        {'facid': 10, 'name': 'Squash Court 2', 'membercost': 3.5,
         'guestcost': 17.5, 'initialoutlay': 5000, 'monthlymaintenance': 80}]
    res = Facility.insert_many(data).execute()


データの計算結果をテーブルに insert する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

facilities テーブルに再度 spa を追加してみましょう.ただこの時,次の facid 
を定数として追加するのではなく,値を自動生成したいと思います.facid 以外は
以下の値を使います:
Name: 'Spa', membercost: 20, guestcost: 30, initialoutlay: 100000,
monthlymaintenance: 800.

.. code-block:: sql

    INSERT INTO "facilities" ("facid", "name", "membercost", "guestcost",
      "initialoutlay", "monthlymaintenance")
    SELECT (SELECT (MAX("facid") + 1) FROM "facilities") AS _,
            'Spa', 20, 30, 100000, 800;

.. code-block:: python

    maxq = Facility.select(fn.MAX(Facility.facid) + 1)
    subq = Select(columns=(maxq, 'Spa', 20, 30, 100000, 800))
    res = Facility.insert_from(subq, Facility._meta.sorted_fields).execute()

既存のいくつかのデータを update する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

２つ目のテニスコートのデータを入れる際に間違えてしまいました.初期費用は 8000
ではなく 10000 でした: あなたはデータを変更してエラーを修正する必要があります.

.. code-block:: sql

    UPDATE facilities SET initialoutlay = 10000 WHERE name = 'Tennis Court 2';

.. code-block:: python

    res = (Facility
           .update({Facility.initialoutlay: 10000})
           .where(Facility.name == 'Tennis Court 2')
           .execute())

    # OR:
    res = (Facility
           .update(initialoutlay=10000)
           .where(Facility.name == 'Tennis Court 2')
           .execute())

複数行と複数カラムを同時に update する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

私達はメンバーとゲスト双方について,テニスコートの値段を上げたいと思います.
メンバーの料金は 6,ゲストは 30 になるように更新してください.

.. code-block:: sql

    UPDATE facilities SET membercost=6, guestcost=30 WHERE name ILIKE 'Tennis%';

.. code-block:: python

    nrows = (Facility
             .update(membercost=6, guestcost=30)
             .where(Facility.name.startswith('Tennis'))
             .execute())

別の行の内容に基づいて行を update する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2番目のテニスコートの価格を,1番目の価格の 10% 増しにしたいと思います.必要に
応じてそのステートメントを再利用できるように,これを定数を使わずにやってみてください.

.. code-block:: sql

    UPDATE facilities SET
    membercost = (SELECT membercost * 1.1 FROM facilities WHERE facid = 0),
    guestcost = (SELECT guestcost * 1.1 FROM facilities WHERE facid = 0)
    WHERE facid = 1;

    -- または --
    WITH new_prices (nmc, ngc) AS (
      SELECT membercost * 1.1, guestcost * 1.1
      FROM facilities WHERE name = 'Tennis Court 1')
    UPDATE facilities
    SET membercost = new_prices.nmc, guestcost = new_prices.ngc
    FROM new_prices
    WHERE name = 'Tennis Court 2'

.. code-block:: python

    sq1 = Facility.select(Facility.membercost * 1.1).where(Facility.facid == 0)
    sq2 = Facility.select(Facility.guestcost * 1.1).where(Facility.facid == 0)

    res = (Facility
           .update(membercost=sq1, guestcost=sq2)
           .where(Facility.facid == 1)
           .execute())

    # または:
    cte = (Facility
           .select(Facility.membercost * 1.1, Facility.guestcost * 1.1)
           .where(Facility.name == 'Tennis Court 1')
           .cte('new_prices', columns=('nmc', 'ngc')))
    res = (Facility
           .update(membercost=SQL('new_prices.nmc'), guestcost=SQL('new_prices.ngc'))
           .with_cte(cte)
           .from_(cte)
           .where(Facility.name == 'Tennis Court 2')
           .execute())

すべての予約を削除する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

データベースの掃除の一環として,bookings テーブルからすべての予約データを
削除したいと思います.

.. code-block:: sql

    DELETE FROM bookings;

.. code-block:: python

    nrows = Booking.delete().execute()


cd.members テーブルからメンバーを削除する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

データベースから,一切予約をしていない 37 番のメンバーを削除します.

.. code-block:: sql

    DELETE FROM members WHERE memid = 37;

.. code-block:: python

    nrows = Member.delete().where(Member.memid == 37).execute()

サブクエリー基づいて削除する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

より汎用的に,一切予約をしていないメンバーをすべて削除するにはどうすればよいでしょうか？

.. code-block:: sql

    DELETE FROM members WHERE NOT EXISTS (
      SELECT * FROM bookings WHERE bookings.memid = members.memid);

.. code-block:: python

    subq = Booking.select().where(Booking.member == Member.memid)
    nrows = Member.delete().where(~fn.EXISTS(subq)).execute()


集約関数
-----------

集約関数は,リレーショナルデータベースシステムのパワーの恩恵を受けられる機能のひとつです.
これにより,単にデータを保持するだけというレベルから,意思決定にも役立てられるような
本当に興味のある質問を投げかけることができるようになります.このカテゴリーでは
集約関数を詳細にカバーします.標準的なグルーピングに加えて,最新のウィンドウ関数に
ついても言及します.


設備の数をカウントする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

集約関数への最初の入口として,まずは簡単な例からご紹介します.施設がどれだけ存在
するのかを知るにはどうすればよいでしょうか？ - 単に合計をカウントしてみましょう.

.. code-block:: sql

    SELECT COUNT(facid) FROM facilities;

.. code-block:: python

    query = Facility.select(fn.COUNT(Facility.facid))
    count = query.scalar()

    # または:
    count = Facility.select().count()

高価な設備の数をカウントする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ゲストの料金が 10 以上である施設の数をカウントしてみます.

.. code-block:: sql

    SELECT COUNT(facid) FROM facilities WHERE guestcost >= 10

.. code-block:: python

    query = Facility.select(fn.COUNT(Facility.facid)).where(Facility.guestcost >= 10)
    count = query.scalar()

    # または:
    # count = Facility.select().where(Facility.guestcost >= 10).count()

それぞれのメンバーが行った被推薦者のカウント
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

それぞれのメンバーが行った被推薦者をカウントしてみます.結果はメンバー ID でソートします.

.. code-block:: sql

    SELECT recommendedby, COUNT(memid) FROM members
    WHERE recommendedby IS NOT NULL
    GROUP BY recommendedby
    ORDER BY recommendedby

.. code-block:: python

    query = (Member
             .select(Member.recommendedby, fn.COUNT(Member.memid))
             .where(Member.recommendedby.is_null(False))
             .group_by(Member.recommendedby)
             .order_by(Member.recommendedby))


施設ごとの予約時間枠の合計一覧を取得する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設ごとの予約時間枠の合計一覧を求めてみましょう.現時点では単に,施設 ID と
時間枠で構成されるテーブルを施設 ID でソートしたものを出力してみます.

.. code-block:: sql

    SELECT facid, SUM(slots) FROM bookings GROUP BY facid ORDER BY facid;

.. code-block:: python

    query = (Booking
             .select(Booking.facid, fn.SUM(Booking.slots))
             .group_by(Booking.facid)
             .order_by(Booking.facid))


指定月の施設ごとの予約時間枠の合計一覧を取得する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012年9月の,施設ごとの予約時間枠の合計一覧を求めてみましょう.施設 ID と時間枠
で構成されるテーブルを,時間枠の数でソートしたものを出力してみます.

.. code-block:: sql

    SELECT facid, SUM(slots)
    FROM bookings
    WHERE (date_trunc('month', starttime) = '2012-09-01'::dates)
    GROUP BY facid
    ORDER BY SUM(slots)

.. code-block:: python

    query = (Booking
             .select(Booking.facility, fn.SUM(Booking.slots))
             .where(fn.date_trunc('month', Booking.starttime) == datetime.date(2012, 9, 1))
             .group_by(Booking.facility)
             .order_by(fn.SUM(Booking.slots)))

月ごとの施設ごとの予約時間枠の合計一覧を取得する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012年の,月ごと施設ごとの予約時間枠の合計一覧を求めてみましょう.施設 ID と時間枠
で構成されるテーブルを,施設 ID と月でソートしたものを出力してみます.

.. code-block:: sql

    SELECT facid, date_part('month', starttime), SUM(slots)
    FROM bookings
    WHERE date_part('year', starttime) = 2012
    GROUP BY facid, date_part('month', starttime)
    ORDER BY facid, date_part('month', starttime)

.. code-block:: python

    month = fn.date_part('month', Booking.starttime)
    query = (Booking
             .select(Booking.facility, month, fn.SUM(Booking.slots))
             .where(fn.date_part('year', Booking.starttime) == 2012)
             .group_by(Booking.facility, month)
             .order_by(Booking.facility, month))


最低１件の予約を行ったメンバーの数を算出する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最低１件の予約を行ったメンバー数の合計を算出してみましょう.

.. code-block:: sql

    SELECT COUNT(DISTINCT memid) FROM bookings

    -- または --
    SELECT COUNT(1) FROM (SELECT DISTINCT memid FROM bookings) AS _

.. code-block:: python

    query = Booking.select(fn.COUNT(Booking.member.distinct()))

    # または:
    query = Booking.select(Booking.member).distinct()
    count = query.count()  # count() wraps in SELECT COUNT(1) FROM (...)

予約された時間枠が1000個以上となる施設の一覧
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

予約された時間枠が1000個以上となるような施設の一覧を求めてみましょう.施設 ID
と時間により構成されるテーブルを,施設 ID でソートしたものを出力してみます.

.. code-block:: sql

    SELECT facid, SUM(slots) FROM bookings
    GROUP BY facid
    HAVING SUM(slots) > 1000
    ORDER BY facid;

.. code-block:: python

    query = (Booking
             .select(Booking.facility, fn.SUM(Booking.slots))
             .group_by(Booking.facility)
             .having(fn.SUM(Booking.slots) > 1000)
             .order_by(Booking.facility))

施設ごとの売上合計を求める
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設の一覧を,それらの売上合計と合わせて求めてみましょう.出力されるテーブルは
施設名と売上により構成され,売上でソートされています.ゲストとメンバーでは
利用料が異なることに注意してください。


.. code-block:: sql

    SELECT f.name, SUM(b.slots * (
    CASE WHEN b.memid = 0 THEN f.guestcost ELSE f.membercost END)) AS revenue
    FROM bookings AS b
    INNER JOIN facilities AS f ON b.facid = f.facid
    GROUP BY f.name
    ORDER BY revenue;

.. code-block:: python

    revenue = fn.SUM(Booking.slots * Case(None, (
        (Booking.member == 0, Facility.guestcost),
    ), Facility.membercost))

    query = (Facility
             .select(Facility.name, revenue.alias('revenue'))
             .join(Booking)
             .group_by(Facility.name)
             .order_by(SQL('revenue')))


売上合計が1000未満の施設の一覧
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

売上合計が1000未満の施設の一覧を求めてみます.出力は施設名と売上により構成され,
売上によりソートされます.ゲストとメンバーでは利用料が異なることに注意してください。

.. code-block:: sql

    SELECT f.name, SUM(b.slots * (
    CASE WHEN b.memid = 0 THEN f.guestcost ELSE f.membercost END)) AS revenue
    FROM bookings AS b
    INNER JOIN facilities AS f ON b.facid = f.facid
    GROUP BY f.name
    HAVING SUM(b.slots * ...) < 1000
    ORDER BY revenue;

.. code-block:: python

    # Same definition as previous example.
    revenue = fn.SUM(Booking.slots * Case(None, (
        (Booking.member == 0, Facility.guestcost),
    ), Facility.membercost))

    query = (Facility
             .select(Facility.name, revenue.alias('revenue'))
             .join(Booking)
             .group_by(Facility.name)
             .having(revenue < 1000)
             .order_by(SQL('revenue')))

予約された時間枠数が最も多い施設 ID を出力する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

予約された時間枠数が最も多い施設 ID を出力します.

.. code-block:: sql

    SELECT facid, SUM(slots) FROM bookings
    GROUP BY facid
    ORDER BY SUM(slots) DESC
    LIMIT 1

.. code-block:: python

    query = (Booking
             .select(Booking.facility, fn.SUM(Booking.slots))
             .group_by(Booking.facility)
             .order_by(fn.SUM(Booking.slots).desc())
             .limit(1))

    # Retrieve multiple scalar values by calling scalar() with as_tuple=True.
    facid, nslots = query.scalar(as_tuple=True)

月ごと施設ごとの予約時間枠の合計一覧を取得する - part 2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012年の月ごと施設ごとの予約時間枠の合計の一覧を取得します.このバージョンでは,
出力行に施設ごとのすべての月の合計と,施設全体のすべての月の合計が現れます.
出力テーブルは施設 ID, 月, 時間枠により構成され,施設 ID と月でソートされます.
すべての月とすべての施設の集約値を計算する際は,月と施設 ID は null 値を返します.

※ Postgres のみ.

.. code-block:: sql

    SELECT facid, date_part('month', starttime), SUM(slots)
    FROM booking
    WHERE date_part('year', starttime) = 2012
    GROUP BY ROLLUP(facid, date_part('month', starttime))
    ORDER BY facid, date_part('month', starttime)

.. code-block:: python

    month = fn.date_part('month', Booking.starttime)
    query = (Booking
             .select(Booking.facility,
                     month.alias('month'),
                     fn.SUM(Booking.slots))
             .where(fn.date_part('year', Booking.starttime) == 2012)
             .group_by(fn.ROLLUP(Booking.facility, month))
             .order_by(Booking.facility, month))


施設ごとの予約合計時間のリスト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

時間枠は 30 分単位であることを考慮しつつ,施設ごとの予約時間の合計の一覧を求めます.
出力テーブルは施設 ID, 施設名, 予約時間からなり,施設 ID でソートされます.

.. code-block:: sql

    SELECT f.facid, f.name, SUM(b.slots) * .5
    FROM facilities AS f
    INNER JOIN bookings AS b ON (f.facid = b.facid)
    GROUP BY f.facid, f.name
    ORDER BY f.facid

.. code-block:: python

    query = (Facility
             .select(Facility.facid, Facility.name, fn.SUM(Booking.slots) * .5)
             .join(Booking)
             .group_by(Facility.facid, Facility.name)
             .order_by(Facility.facid))


2012/09/01 以降に最初に予約したメンバーのリスト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

2012/09/01 以降最初に予約したメンバーについて,メンバー名,メンバー ID
および予約開始時間のリストを求めます.ソート順はメンバー ID です.

.. code-block:: sql

    SELECT m.surname, m.firstname, m.memid, min(b.starttime) as starttime
    FROM members AS m
    INNER JOIN bookings AS b ON b.memid = m.memid
    WHERE starttime >= '2012-09-01'
    GROUP BY m.surname, m.firstname, m.memid
    ORDER BY m.memid;

.. code-block:: python

    query = (Member
             .select(Member.surname, Member.firstname, Member.memid,
                     fn.MIN(Booking.starttime).alias('starttime'))
             .join(Booking)
             .where(Booking.starttime >= datetime.date(2012, 9, 1))
             .group_by(Member.surname, Member.firstname, Member.memid)
             .order_by(Member.memid))

各行にメンバー数の合計を記載したメンバー名の一覧
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

各行にメンバー数の合計を含むメンバー名の一覧です.ソートキーは加入日です.

Postgres のみ (文字通り).

.. code-block:: sql

    SELECT COUNT(*) OVER(), firstname, surname
    FROM members ORDER BY joindate

.. code-block:: python

    query = (Member
             .select(fn.COUNT(Member.memid).over(), Member.firstname,
                     Member.surname)
             .order_by(Member.joindate))

メンバーのリストに番号を振ったもの
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メンバー一覧に対して単純に昇順の番号を振り,加入日でソートしたものを求めます.
メンバー ID は昇順になっているとは限らないことに注意してください.

Postgres のみ (文字通り).

.. code-block:: sql

    SELECT row_number() OVER (ORDER BY joindate), firstname, surname
    FROM members ORDER BY joindate;

.. code-block:: python

    query = (Member
             .select(fn.row_number().over(order_by=[Member.joindate]),
                     Member.firstname, Member.surname)
             .order_by(Member.joindate))

再度,予約時間枠数が最大の施設 ID を出力
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

予約時間枠数が最大である施設の ID を求めます.複数あった場合はそれらが
すべて出力されます.

Postgres のみ (文字通り).

.. code-block:: sql

    SELECT facid, total FROM (
      SELECT facid, SUM(slots) AS total,
             rank() OVER (order by SUM(slots) DESC) AS rank
      FROM bookings
      GROUP BY facid
    ) AS ranked WHERE rank = 1

.. code-block:: python

    rank = fn.rank().over(order_by=[fn.SUM(Booking.slots).desc()])

    subq = (Booking
            .select(Booking.facility, fn.SUM(Booking.slots).alias('total'),
                    rank.alias('rank'))
            .group_by(Booking.facility))

    # Here we use a plain Select() to create our query.
    query = (Select(columns=[subq.c.facid, subq.c.total])
             .from_(subq)
             .where(subq.c.rank == 1)
             .bind(db))  # We must bind() it to the database.

    # クエリー結果をイテレートする場合:
    for facid, total in query.tuples():
        print(facid, total)

利用時間(概数)によるメンバーのランキング
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設を予約した時間を10時間単位で四捨五入した数とともに,メンバー一覧を出力します.
この概算値を使って順位付けを行い,名,姓,概算時間数,順位を出力します.ソート順は
順位,姓,名です.

Postgres のみ (文字通り).

.. code-block:: sql

    SELECT firstname, surname,
    ((SUM(bks.slots)+10)/20)*10 as hours,
    rank() over (order by ((sum(bks.slots)+10)/20)*10 desc) as rank
    FROM members AS mems
    INNER JOIN bookings AS bks ON mems.memid = bks.memid
    GROUP BY mems.memid
    ORDER BY rank, surname, firstname;

.. code-block:: python

    hours = ((fn.SUM(Booking.slots) + 10) / 20) * 10
    query = (Member
             .select(Member.firstname, Member.surname, hours.alias('hours'),
                     fn.rank().over(order_by=[hours.desc()]).alias('rank'))
             .join(Booking)
             .group_by(Member.memid)
             .order_by(SQL('rank'), Member.surname, Member.firstname))


収入合計がトップ３の施設
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

トップ３の売上を計上した施設の一覧を求めます(同順位も考慮).出力は施設名と
順位により構成され,順位と施設名でソートされます.

Postgres のみ (文字通り).

.. code-block:: sql

    SELECT name, rank FROM (
        SELECT f.name, RANK() OVER (ORDER BY SUM(
            CASE WHEN memid = 0 THEN slots * f.guestcost
            ELSE slots * f.membercost END) DESC) AS rank
        FROM bookings
        INNER JOIN facilities AS f ON bookings.facid = f.facid
        GROUP BY f.name) AS subq
    WHERE rank <= 3
    ORDER BY rank;

.. code-block:: python

   total_cost = fn.SUM(Case(None, (
       (Booking.member == 0, Booking.slots * Facility.guestcost),
   ), (Booking.slots * Facility.membercost)))

   subq = (Facility
           .select(Facility.name,
                   fn.RANK().over(order_by=[total_cost.desc()]).alias('rank'))
           .join(Booking)
           .group_by(Facility.name))

   query = (Select(columns=[subq.c.name, subq.c.rank])
            .from_(subq)
            .where(subq.c.rank <= 3)
            .order_by(subq.c.rank)
            .bind(db))  # Here again we used plain Select, and call bind().

値により施設を分類する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

施設をそれらの売上額の高い(high),平均(average),低い(low)という３グループに分類します.
ソート順は分類と施設名です.


Postgres のみ (文字通り).

.. code-block:: sql

    SELECT name,
      CASE class WHEN 1 THEN 'high' WHEN 2 THEN 'average' ELSE 'low' END
    FROM (
      SELECT f.name, ntile(3) OVER (ORDER BY SUM(
        CASE WHEN memid = 0 THEN slots * f.guestcost ELSE slots * f.membercost
        END) DESC) AS class
      FROM bookings INNER JOIN facilities AS f ON bookings.facid = f.facid
      GROUP BY f.name
    ) AS subq
    ORDER BY class, name;

.. code-block:: python

    cost = fn.SUM(Case(None, (
        (Booking.member == 0, Booking.slots * Facility.guestcost),
    ), (Booking.slots * Facility.membercost)))
    subq = (Facility
            .select(Facility.name,
                    fn.NTILE(3).over(order_by=[cost.desc()]).alias('klass'))
            .join(Booking)
            .group_by(Facility.name))

    klass_case = Case(subq.c.klass, [(1, 'high'), (2, 'average')], 'low')
    query = (Select(columns=[subq.c.name, klass_case])
             .from_(subq)
             .order_by(subq.c.klass, subq.c.name)
             .bind(db))

再帰
---------

クエリーの実行中に,一般的なテーブル表現を使って効率的に中間的なテーブルを生成
することができます - それらは SQL をより読みやすくしてくれるので、非常に便利な
ものです.しかしながら, WITH RECURSIVE 修飾子を使うと,再帰クエリーを生成する
ことができます.これはツリー構造やグラフ構造のデータを処理する際,とても都合の
よいものです - たとえば,指定された深度でグラフノードのリレーション全体を探索
することを考えてみてください.


メンバー ID 27 についての推薦チェーンで上昇中のものを見つける
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メンバー ID 27 について,推薦チェーンの中から上昇中のものを見つけることを考えます:
すなわち,他の誰かを推薦したメンバーと,そのメンバーを推薦したメンバーを見つける
といったことです.このクエリーではメンバー ID,名,姓を返します.ソート順はメンバー
ID の降順です.

.. code-block:: sql

    WITH RECURSIVE recommenders(recommender) as (
      SELECT recommendedby FROM members WHERE memid = 27
      UNION ALL
      SELECT mems.recommendedby
      FROM recommenders recs
      INNER JOIN members AS mems ON mems.memid = recs.recommender
    )
    SELECT recs.recommender, mems.firstname, mems.surname
    FROM recommenders AS recs
    INNER JOIN members AS mems ON recs.recommender = mems.memid
    ORDER By memid DESC;

.. code-block:: python

    # 再帰 CTE の基本ケース.memid=27 の推薦者を見つける.
    base = (Member
            .select(Member.recommendedby)
            .where(Member.memid == 27)
            .cte('recommenders', recursive=True, columns=('recommender',)))

    # CTE の再帰部分.直近の推薦者の推薦者を見つける.
    MA = Member.alias()
    recursive = (MA
                 .select(MA.recommendedby)
                 .join(base, on=(MA.memid == base.c.recommender)))

    # Combine the base-case with the recursive term.
    # 基本ケースと再帰部分を結合
    cte = base.union_all(recursive)

    # 再帰 CTE から select し,member と結合して名前の情報を取得
    query = (cte
             .select_from(cte.c.recommender, Member.firstname, Member.surname)
             .join(Member, on=(cte.c.recommender == Member.memid))
             .order_by(Member.memid.desc()))
