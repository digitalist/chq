# 0. ClickHouse unofficial FAQ. Intro

[Russian version](README.md)

This file is an ongoing compilation from our own errors, questions and edited copy-paste from [ClickHouse telegram channel](https://t.me/clickhouse_ru).
Improvements and PR's are welcome

# 1. Building/installing/running

## 1.0 tl;dr: connection

CH binds on two ports:
- 8123 (http),
- 9000 (binary protocol).

You won't be able to connect if you mix them up. Don't mix them in your apps or ssh tunnels.

## 1.1 config:
the maing config is here:

`/etc/clickhouse-server/config.xml`
    
main options:

- listener (what interface/network card to listen,  0.0.0.0 to listen on all interfaces, if you doon't care for security or running it inside virtual network only)
- <path>/var/lib/clickhouse/</path>: path to data

- /etc/metrika.xml (default filename) you can override some options:
    <yandex><zookeeper><node id='1'><host>172.31.16.254</host><port>2181</port></node></zookeeper></yandex>


## 1.2 distributed build

when building on linux with `./release` script, you need to disable -pie in `debian/rules`, othwerise distcc in its current version
will use only localhost (because of spec- gcc flag, even if you're using clang)

## 1.3 single-node

## 1.4 cluster/replica
    setup goes here
    see 7.2 for questions


## 1.2 single-node

## 1.3 cluster: from official docs:
With default settings Zookeeper is a time bomb. It doesn't dele its logs or snapshots. Take care of it.
https://clickhouse.yandex/docs/ru/operations/tips.html

## 1.4 docker

# 2. работа с clickhouse, общее

## 2.1 Table engins

### MergeTree

[Andrey]
>What engine should I use for storing data that does not depends on date/time? For example a domain list with some properties (~100M records ).
 We need to get various statistics from this data. MergeTree requires EventDate.

[Stanislav Vlasov]
 MergeTree uses partitioning functon, date by default. Documentation contains example of creating tables with a custom keys.

[Alexey Milovidov]
MergeTree doesn't requre partitioning at all. Just don't use PARTITION BY

https://clickhouse.yandex/docs/ru/single/#proizvolnyi-klyuch-particzionirovaniya


# 3. Language bindings and libraries

    - Python: clickhouse_driver, native/binary  python<->c<->clickhouse.
    - Sqlalchemy Clickhouse driver from Cloudflare
    - JDBC: works via http (port 8123)


# 4. Buffering and othir miscellaneous infrastructure

Apache Kafka is the most used option.


>If number of connections between nodes will reach max_distributed_connections, will there be an exception or
queries will be queued?"

With default value 1024, if your cluster has < 1024 shards, you will never hit that limit.
Query to check this value: `select * from system.settings where name = 'max_distributed_connections'.`

See: [proxysql](http://www.proxysql.com/blog/proxysql-143-clickhouse) - for query queueing.

Balancer & proxy [chproxy](https://github.com/Vertamedia/chproxy)


# 5. Working with queries

## 5.0 Important: inserting data

You don't insert small quantites of data into CH. High-frequency inserts of single rows will cause CH dealys, then it will hang. This is caused by inner working of most used types of table engines: *MergeTrees

You can write single rows, but infrequently.

[Tima Ber]
- Do not insert less than 1000 rows. Not 10, not 20, no less than 1к.   
- Insert rows infrequently, no more than once in a second, in a single thread
- It's better to increase batch size instead of using more than one inserting thread 


### 5.0.1 Inserting data using curl/json:

[Alex More]: Use input_format_skip_unknown_fields=1 to ignore  fields present in JSON, but absent in schema

`curl -H "Content-Type: application/json" -X POST -d 'insert into table_name FORMAT JSONEachRow {"create_date": "12345", "ololo":"ololo"}' http://default:psw@localhost:8123/?input_format_skip_unknown_fields=1`


### 5.0.2 Errors during data insert:
#### Be precise!
> `DB::Exception: Element of set in IN or VALUES is not a constant expression`

[Milovidov] 
This errors caused by data being in slightly wrong format. ClickHouse tries to interpret incoming value as an arbitary expression and convert it to needed data type. 
To trace the problem set input_format_values_interpret_expressions to 0

#### And again: When you're using batch inserts, be precise!
[Time Ber] Data can go wrong (shifting columns) when there's absent input
 value but your batch-creating code doesn't fix it. ClickHouse inserts
 default values for every corresponding type.
I.e if there's no date in data, your code should set it to '0000-00-00', or your columns will shift 

### 5.0.2 Buffer engine table vs external buffer app
>Nikita Tokarchuk, [21.02.18 18:46]
> Docs suggest two solutions —  in-app buffer or Buffer engine table

[Andrey @rhenix]: Buffer resides in ClickHouse Memory + overhead from connections. In some cases in-app buffering is better.
If you need a simple 'collect and dump', there's no need to inevnt something, use ClickHouse Buffer.

See documentation on issues related to Buffer + Replicated tables.

### 5.0.3 Moving/restoring data.
All you have to do is stop the server and copy/move data directory (i.e. `/var/lib/clickhouse`) from one machine to another using
any method you like: scp/rsync/mount drive/blue ray snail mail :-)

Accordingly, ClickHouse will see this data in a default location if you (re)install it.

## 5.1 Iterating over data:

### 5.1.1 with some counter_id
counter_id in a primary key:

    SELECT * FROM db.ponylog ORDER BY counter_id LIMIT {lim}, {skip}

Doesn't crash but quickly slows down to unusable speed.

    SELECT * FROM db.ponylog WHERE counter_id between {start} and {end} ORDER BY counter_id

Fastest version

## 5.2 Group by:

- group by column expression: you can't do MySQL-like `select extract(desc, '\\[.*?]') from TBL group by 1` (groupb by first column)
instead you should do:


        SELECT extract(desc, '\\[.*?]')
        FROM bigdb.ponylog
        GROUP BY extract(desc, '\\[.*?]')

## 5.3 String search:

string = # match

    SELECT count(*)
    FROM bigdb.ponylog
    WHERE desc = '[Pony Updater] Update from VK:<br/>Status: archived - paused<br/>'
    LIMIT 1

    Row 1:
    ──────
    count(): 20190018

    1 rows in set. Elapsed: 0.782 sec. Processed 38.96 million rows, 3.93 GB (49.80 million rows/s., 5.03 GB/s.)

string like '%pattern%'  # wildcard match

    SELECT count(*)
    FROM bigdb.ponylog
    WHERE desc LIKE '% Updater%'

    ┌──count()─┐
    │ 20617731 │
    └──────────┘

    1 rows in set. Elapsed: 0.832 sec. Processed 38.96 million rows, 3.93 GB (46.81 million rows/s., 4.73 GB/s.)

string with regexp/group by regexp

    SELECT
        count(*),
        extract(desc, '\\[.*?]')
    FROM bigdb.ponylog
    GROUP BY extract(desc, '\\[.*?]');

    ┌──count()─┬─extract(desc, \'\\\\[.*?]\')─┐
    │  1078529 │                              │
    │   556649 │ [StatusChanger]              │


## 5.4 External dictionaries

### External dictionaries use cases, MySQL example 


While using join you have to load data for *every* query. Using dictionary avoids it by placing all needed data in-memory once
and syncing it on a regular basis

[shegloff]

>Suppose we have ClickHouse table BigDataBannerShows.LogsAggregated
We put business logic dictionary values (like user_options from UserDictionary.UserInfo, status from Partners.Site and so on  ) 
into a `where` clause. Those  are tables in MySQL and to avoid copying them to ClickHouse we plug them in as an external dictionaries,
auto-syncing every 5 minutes..

> ClickHouse doesn't ask MySQL directly on per-request basis, instead it selects all needed data every 5 minutes
and keeps it in memory.

>You don't have to keep entire dictionary in memory, there's an option to keep a limited cache, leaving full data in, for example, MySQL [Yuran aka yourock88]

>How to write a WHERE/ON clause on the column from external table?
- Using dictionary and `getDict()`. [Vasilij Abrosimov]


## 5.5 Working with joins

If you're doing dumb queries with very big tables, during joins CH can pull one of the tables in memory and it can be too much.
> dumb query: `ANY INNER JOIN urls2 USING id`

let's constrain it with subquery and `where` clause:

>`ANY INNER JOIN (SELECT id, url FROM urls2 WHERE id = url_id) USING id` [Tima Ber]
or
>`ANY INNER JOIN (SELECT id, count() AS shows FROM urls2 WHERE day = today) USING (id)` [Shegloff]

## collapsingMergeTree related (?)
вывести предыдущее значение строки
event - runningDifference(event) [Владимир Мюге]

## 5.6 Разные вопросы Miscellaneous queries

### 5.6.1 Missing date values
> подскажите а есть ли способ при выборке count() с группировкой по дате получить
> 0 для пустых дней, вместо пропущенных строк в результате? вместо

    20180102 1
    20180104 1
    получить
    20180102 1
    20180103 0
    20180104 1

[Alexey Sheglov] Немного наркомании

    :) select day, count() from test.huest group by day

    ┌────────day─┬─count()─┐
    │ 2018-02-21 │       1 │
    │ 2018-03-03 │       1 │
    └────────────┴─────────┘

    :) select day, sum(c) as count from (select day, count() as c from test.huest group by day) ALL RIGHT JOIN (select day, c from (select toDate('1970-01-01') + number as day, 0 as c from system.numbers limit 30000) where day between (select min(day) from test.huest) and (select max(day) from test.huest)) USING(day) group by day

    ┌────────day─┬─count─┐
    │ 2018-02-21 │     1 │
    │ 2018-02-22 │     0 │
    │ 2018-02-23 │     0 │
    │ 2018-02-24 │     0 │
    │ 2018-02-25 │     0 │
    │ 2018-02-26 │     0 │
    │ 2018-02-27 │     0 │
    │ 2018-02-28 │     0 │
    │ 2018-03-01 │     0 │
    │ 2018-03-02 │     0 │
    │ 2018-03-03 │     1 │



### 5.6.1 Array diff

-  I have two arrays inside a subquery, how to get their difference?  Gennadiy Alekseev [@alekseevgena]
 

[Natalya]:

    `SELECT [3, 4, 8] as mas, arrayFilter(x -> x NOT IN (1,2,3,4), mas) AS res`


[Alexey Sheglov / @Shegloff]:

Another slightly perverted solution:

    SELECT
        arrayJoin(one_arr) AS res,
        any(two_arr) AS two_arr
    FROM
    (
        SELECT
            [1, 2, 3, 4, 5] AS one_arr,
            [3, 4, 5, 6, 7, 8] AS two_arr
    )
    GROUP BY res
    HAVING has(two_arr, res) = 0
    
    ┌─res─┬─two_arr───────┐
    │   1 │ [3,4,5,6,7,8] │
    │   2 │ [3,4,5,6,7,8] │
    └─────┴───────────────┘


### 5.6.2 Rotating arrays/columns
- How to convert a column to a string (analogue of concat/group_concat) / rotate + concat column [Qq]

        А
        1
        2
        3
        to: 1,2,3
 

[Nikolai Kochetov]
For columns: you can do `groupArray()`, then join resulting array
For - `arrayStringConcat()`



# 6.  Kafka/Zookeeper:

## 6.1
[использование zk](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.2/bk_kafka-component-guide/content/kafka-zookeeper-multiple-apps.html) для нескольких приложений одновременно (например, кафка+clickhouse)


## 6.2 кафка->кликхауз

[как наливать данные](https://clickhouse.yandex/docs/en/table_engines/kafka.html)

tldr: при работе с кафкой надо создать две таблицы (одна типа кафака, другая, допустим, mergetree) и
1 materialised view, данные принимаются в кафку, а MV в фоне укладывает их в нужную таблицу

[использование кафки через командную строку (для тестов)]( http://cloudurable.com/blog/kafka-tutorial-kafka-from-command-line/index.html):

## 6.3 Troubleshooting

Loading data from MaterialisedView into a table, log messages:
    `StorageKafka (fromkafka): EOF reached for partition 0 offset 59651`

check [stream_flush_interval_ms](https://clickhouse.yandex/docs/en/operations/settings/settings.html)



## 7. Troubleshooting

### 7.1 Inconsistencies during data loading:
[Milovidov]
 
Data insert is always 1:1. If there were no error messages, that is, if  `INSERT SELECT` query was successfull- all data is loaded.
There are cases whith look-alike errors:

- Insertion of the text dump containing new line escaping: 
    
    `abc \

    def`
    
    Raw (`wc -l`) line count will be bigger than real line count
      
- Insertion into a Distributed table, which transfers data in async mode 

- Duplicated blocks of data inserted into ReplicatedMergeTree table will be de-duplicated.

- Writing to a table with an engine which changes data during merge (Collapsing-, Replacing- types)

- Distributed table with incorrect cluster configuration: i.e. `internal_replication = 1` with non-Replicated tables; or when
you mix up shards and replicas

- Writing to a Replicated table and reading from delayed replica 

- Writing to a Buffer table while reading from another type table


### 7.2 Replica

#### Replication setup
Alex [@Player_a], [20.02.18 01:37]
Database with CollapsingMergeTree table, about 100 Gb in size.
How to setup a replication? How does it work? How to send existing data from master?


[Alexey Milovidov], [20.02.18 01:41]
If your table already has ReplicatedCollapsingMergeTree type, it's simple: new replica is added
by `CREATE TABLE` query on the new server.
Set same table path as in ZooKeeper but with another replica ID. After creation replica will sync itself - download all the data it needs;
В параметрах указывается тот же путь таблицы в ZK, но другой идентификатор реплики. После создания, реплика синхронизируется сама -
If your table is CollapsingMergeTree (not Replicated) - see the docs: [Converting from ReplicatedMergeTree to MergeTree](https://clickhouse.yandex/docs/en/table_engines/replication/#converting-from-mergetree-to-replicatedmergetree).


#### Replica troubleshooting
There's little sense looking ZooKeeper's logs, better look CH own logs (there will be something like 'pulling logs to queue'.)
It's simpler to run `select * from system.replicas WHERE database='db' AND table='table' FORMAT Vertical`, related fields are
 `last_queue_update`, `active_replicas`, `absolute_delay`.
BTW To take a look what's going on under the hood, turn on `<part_log>` option in config and check `system.part_log` table.
[Vitaliy Lyudvichenko]


#### Exception: Could not find a column of minimum size in MergeTree
>   `Exception: Could not find a column of minimum size in MergeTree,
>    part /var/lib/clickhouse//data/public/events/20180124_20180124_14437_14442_1/`

There are no such part on this replica. I did `detach partition 201801` and `attach partition 201801`, during attach that lost bit synced from another replica, problem solved

You can delete everything in `detached` directory on a broken replica, and after `attach` it will download entire partition.


#### Inconsistent partition data on a replica node
[Vladislav Denisov]
`Feb 15 16:05:59 cdn2 docker[15331]: 2018.02.15 13:05:59.045028 [ 11 ] <Debug> local.clickstream (StorageReplicatedMergeTree): Part 20180101_20180131_0_267872_24 (state Deleting) should be deleted after previous attempt before fetch`

I noticed inconsitent data, deleted partition and did `scp` copy from the mirrior node, now log file is full of errors like this. Wat do?

[Alexey Sheglov]:

1. `detach partition` - on both nodes parition will be moved to `detached` dir

2. problem_node: `$ rm -rf /path/to/clickhouse_data/detached/*`

3. `attach partition`, problem node will sync partition from healthy node



#### Replica cleanup and deletion

> [Vadim Metikov]
>
> Had to temporary take out a replica from a ClickHouse cluster.
> After adding replicated tables during recreating this replica as `r4`:
> `Code: 253. DB::Exception: Received from localhost:9000, 127.0.0.1. DB::Exception: Replica /clickhouse/tables/01/graphite_tree/replicas/r1 already exists`
> How to delete old `r1` replica?


[Max Pavlov]:  You should delete this replica from ZooKeeper.
It is done automatically if you run `drop database` or `drop table` on an old replica where you have replicated tables
Otherwise you have to add new replica with another id and cleanup zookeeper after that:

`[zk: localhost:2181(CONNECTED) 9] delete /clickhouse/tables/01/graphite/replicas/r1/`
(or use a GUI like zooinspector)


#### Scaling ClickHouse without distributed tables

[Kirill Shvakov]
We just add servers, that's all. Data goes into Kafka, from there we write it do ClickHouse shards, we don't write to distributed tables.
If we need more space or data hits a memory limit, we add a shard.
Accordingly, after adding a shard, data begins to distribute slightly ....???
and due to periodic deletion of old data, shards gradually equate data size

#### ClikHouse  cache (it hasn't one) / becnhmarks

> [Jen Zolotarev]:
> I'd like to test ClickHouse performance with a custom partitioning key vs default key
> using same select queries into a different tables, the problem is CH caches data, blurring benchmark results
> How to clean ClickHouse cache?

[Kirill Shvakov]:
ClickHouse does not have cache. It uses OS cache. Use this:
`SET min_bytes_to_use_direct_io=1, merge_tree_uniform_read_distribution=0`

## 8. Права пользователей, разное.

> [Ilyas Guseynov]:
> Можно ли как-то выставить distributed_group_by_no_merge в единичку для read-only юзеров?
> Есть запрос с группировкой и шардинг гарантирующий размещение этих групп только на конкретных нодах.
> Кажется логичным применять эту настройку, но для read-only оно ругается  "Cannot execute SET query in readonly mode"

[Kirill Shvakov]: readonly в 2 выставить надо (или просто в профиле пользователя указать) - тогда можно делать SET и создавать временные таблицы



#### *. ссылки
##### техническое:
[статья про primary keys в MergeTree таблицах](https://medium.com/@f1yegor/clickhouse-primary-keys-2cf2a45d7324)

[репликация и шардинг](https://www.altinity.com/blog/2017/6/5/clickhouse-data-distribution)
##### около-техническое и применение
[1.1 Billion Taxi Rides on ClickHouse & an Intel Core i5](http://tech.marksblogg.com/billion-nyc-taxi-clickhouse.html)
