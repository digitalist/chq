todo:
ru: format 5.6, add to en

### 2.3 Набросики про запросики
клиент поддерживает довольно приличный sql-подобный язык запросов, примитивные примеры далее.
Много агреггирующих функций, о которых будет рассказано по мере использования.

    зайти в клиент: clickhouse-client

создать простую таблицу:

    create table test.test (dt Date, num UInt32) ENGINE=MergeTree (dt, (num), 8192);
    # у движка MergeTree обязательно должно быть в индексе поле типа Date # ложь


выполнить запрос:

    select  * from test.test; [enter]
    |выдаст | первые 100к рядов | в табличной | форме |

выдать данные в формате `ключ: значение` (как в mysql):

    select  * from db.table\G [enter]


мониторинг через клиента (разобраться дальше):

    select * from system.asynchronous_metrics;
    select * from system.events;


убить запрос/долгоиграющий запрос: выяснить.


работа с кластерами:
create table не реплицируется
create table нет, но  есть create table ... on cluster
синхронизация /var/lib/clickhouse/metadata происходит копированием данных

#### узнать тип данных возвращаемой функции
`SELECT toTypeName(IPv4StringToNum(''))` [papa carlo]


Vadim Metikov, [26.02.18 13:59]
кто-нибудь прореживал так данные? партишионинг по ГГГГММ более всего подходит


Alexey Sheglov
Иван
подскажите а есть ли способ при выборке count() с группировкой по дате получить 0 для пустых дней, вместо пропущенных строк в результате?
вместо
20180102 1
20180104 1
получить 
20180102 1
20180103 0
20180104 1

Немного наркомании

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


Кто-нибудь может пояснить, зачем Clickhouse для ReplicatedMergeTree в режиме 3 шарда по 2 реплики создаёт на каждом серврере какие-то бинарники в папках с хостами каждой ноды, и вес этих бинарников больше импортируемого несжатого файла с данными?
Сжатие включено zstd на каждой из 6 нод, был на одну ноду в Distibuted импортирован файл 10G.

root@ch-node-4:/var/lib/clickhouse/data/distributedbd/ppitest_datagate# ls -l
total 144304
drwxr-xr-x 3 clickhouse clickhouse 33476608 Mar  4 05:34 default@192%2E168%2E10%2E129:9000
drwxr-xr-x 3 clickhouse clickhouse 28807168 Mar  4 05:34 default@192%2E168%2E10%2E130:9000
drwxr-xr-x 2 clickhouse clickhouse 27439104 Mar  4 05:34 default@192%2E168%2E10%2E132:9000
drwxr-xr-x 3 clickhouse clickhouse 29003776 Mar  4 05:34 default@192%2E168%2E10%2E134:9000
drwxr-xr-x 2 clickhouse clickhouse 28667904 Mar  4 05:34 default@192%2E168%2E10%2E135:9000
root@ch-node-4:/var/lib/clickhouse/data/distributedbd/ppitest_datagate# du . -hx --max-depth=1 2> /dev/null
4.9G ./default@192%2E168%2E10%2E134:9000
34M ./default@192%2E168%2E10%2E135:9000
5.7G ./default@192%2E168%2E10%2E129:9000
4.6G ./default@192%2E168%2E10%2E132:9000
273M ./default@192%2E168%2E10%2E130:9000
16G .
ppitest_datagate - имя таблицы Distributed. Как-то плюс очень активно кушаются inodes, это напрягает.


Решилось установкой параметра internal_replication в true для шард и рестартом серверов. В документации очень блекло написано и сходу непонятно, к чему этот параметр и чем он грозит, когда Destributed пишет в Replicated, пока на практике не столкнёшься...
