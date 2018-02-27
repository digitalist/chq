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