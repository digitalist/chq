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


Stanislav Vlasov, [06.03.18 17:27]
ALTER TABLE ... DROP PARTITION ...

Stanislav Vlasov, [06.03.18 17:27]
Имена можно подсмотреть в system.parts

Stanislav Vlasov, [06.03.18 17:28]
Что-то типа:
SELECT partition FROM system.parts WHERE (database = '${DB}') AND (table = '${TB}') AND active GROUP BY partition ORDER BY partition ASC