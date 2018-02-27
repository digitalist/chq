# 0. Intro

Дока формируется на основе получения личных шишек, консультаций и редактированной копипасты с https://t.me/clickhouse_ru.
Дополнения приветствуются.

# 1. сборка и запуск

## 1.0 tl;dr: подключение

CH binds on two ports
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
    - JDBC: works via http (port 8123)




# 4. Buffering and othir miscellaneous infrastructure

Apache Kafka is the most used option


[proxysql](http://www.proxysql.com/blog/proxysql-143-clickhouse) - для реализации механизма очереди запросов -
например "Если число соединений между серверами будет больше чем max_distributed_connections,
выкинется исключение или запросы встанут в очередь?"
С дефолтным значением 1024 и если у вас в кластере меньше 1024 шардов, вы в этот лимит никогда не упрётесь.
Значение можете проверить запросом select * from system.settings where name = 'max_distributed_connections'.

балансировщик и прокси [chproxy](https://github.com/Vertamedia/chproxy)


# 5. Working with queries

## 5.0 Important: inserting data

You don't insert small quantites of data into CH. High-frequency inserts of single rows will cause CH dealys, then it will hang. This is caused by inner working of most used types of table engines: *MergeTrees

You can write single rows, but infrequently.

[Tima Ber]
- Do not insert less than 1000 rows. Not 10, not 20, no less than 1к.   
- Insert rows infrequently, no more than once in a second, in a single thread
- It's better to increase batch size instead of using more than one inserting thread 


### 5.0.1 Inserting data usingcurl/json:

[Alex More]: Use input_format_skip_unknown_fields=1 to ignore  fields present in JSON, but absent in schema

`curl -H "Content-Type: application/json" -X POST -d 'insert into table_name FORMAT JSONEachRow {"create_date": "12345", "ololo":"ololo"}' http://default:psw@localhost:8123/?input_format_skip_unknown_fields=1`


### 5.0.2 Errors during data insert:
#### Be precise!
> `DB::Exception: Element of set in IN or VALUES is not a constant expression`

[Milovidov] Cвязано с тем, что данные для вставки записаны не в точно ожидаемом формате. Тогда ClickHouse пытается интерпретировать указанное значение как произвольное выражение и затем преобразовать его к нужному типу данных.
Чтобы увидеть более явно, где возникает проблема, выставите настройку input_format_values_interpret_expressions в ноль.

#### Еще раз: в батчах тоже вставляйте данные точно!
[Time Ber] Данные могут поехать, когда в исходных данных нет значения, а коде формирования батча не учитывается это и не проставляются дефолтные значения для каждого из типов.
Например в данных нет даты, тогда в строку для КХ нужно ставить '0000-00-00', иначе при вставке столбцы "поедут"



### 5.0.2 таблица типа Buffer vs buffer-приложение
>Nikita Tokarchuk, [21.02.18 18:46]
>В чем может быть плюс своего решения против Buffer таблицы?
> Цель — буферизировать запросы на вставку, идущие в реалтайме один за другим.
>
> Документация предлагает два решения — делать буфер в приложении которое делает insert, либо использовать Buffer таблицу

[Andrey @rhenix]: Buffer лежит в оперативке + расход на коннекты от разных инсертов. Поэтому в некоторых случаях буферизация на аппликухе куда лучше

Если нужно чтобы данные собирались и интервально сбрасывались, то нет смысла изобретать велосипед, Buffer для того и живет

### 5.0.3 Moving/restoring data.

Для переноса данных достаточно остановить сервер и перенести директорию с данными (например /var/lib/clickhouse) с одной машины на другую любым
удобным способом - scp/rsync/mount hdd/ssd :-)

Соответственно, и при переустановке самого кликхауза данные подхватятся автоматически.


[Salim Murtazaliev]
Если таблица назначения является реплицируемой, то при записи в таблицу Buffer будут потеряны некоторые ожидаемые свойства реплицируемых таблиц. Из-за произвольного изменения порядка строк и
размеров блоков данных, перестаёт работать дедупликация данных, в результате чего исчезает возможность надёжной exactly once записи в реплицируемые таблицы.

## 5.1 Итерация по базе:

### 5.1.1 использование с counter_id
counter_id находится в составе primary key:

запрос 1:

    SELECT * FROM vkad.ponylog WHERE counter_id between {lim} and {skip}  ORDER BY counter_id

без лимита по памяти на запрос (max_memory_usage в /etc/clichkouse-server/user.xml) -  по дефолту 10 гб - уходит в своп
с лимитом 2 гб.  - падает на ~10-м запросе. работает медленно

запрос 2:

    SELECT * FROM vkad.ponylog ORDER BY counter_id LIMIT {lim}, {skip}
не падает по памяти, но очень быстро замедляется до неприемлемых значений, при этом нет сортировки.

запрос 3:

    SELECT * FROM vkad.ponylog WHERE counter_id between {start} and {end} ORDER BY counter_id

летает отлично, быстро, сортировка работает.

поскольку нормального explain нет, выглядит это так, что в первом запросе кликхауз пожирает всю доступную память, видимо многократно запихивая индекс в память.
руководство о таком предупреждает, но, конечно, нам это не подходит


## 5.2 Группировки:

- с помощью функций: в отличие от mysql нельзя сделать запрос в духе `select extract(desc, '\\[.*?]') from TBL group by 1` (сгруппировать по первой колонке)
нужно делать так:


        SELECT extract(desc, '\\[.*?]')
        FROM bigdb.ponylog
        GROUP BY extract(desc, '\\[.*?]')

## 5.3 Поиск по строкам:

string = # точное совпадение

    SELECT count(*)
    FROM bigdb.ponylog
    WHERE desc = '[Pony Updater] Update from VK:<br/>Status: archived - paused<br/>'
    LIMIT 1

    Row 1:
    ──────
    count(): 20190018

    1 rows in set. Elapsed: 0.782 sec. Processed 38.96 million rows, 3.93 GB (49.80 million rows/s., 5.03 GB/s.)

string like '%pattern%'  #частичное совпадение

    SELECT count(*)
    FROM bigdb.ponylog
    WHERE desc LIKE '% Updater%'

    ┌──count()─┐
    │ 20617731 │
    └──────────┘

    1 rows in set. Elapsed: 0.832 sec. Processed 38.96 million rows, 3.93 GB (46.81 million rows/s., 4.73 GB/s.)

string with regexp/group by regexp #поиск по регулярному выражению, с группировкой по нему же

    SELECT
        count(*),
        extract(desc, '\\[.*?]')
    FROM bigdb.ponylog
    GROUP BY extract(desc, '\\[.*?]');

    ┌──count()─┬─extract(desc, \'\\\\[.*?]\')─┐
    │  1078529 │                              │
    │   556649 │ [StatusChanger]              │


## 5.4 External dictionaries

External dictionaries use cases, MySQL example [shegloff]

>Suppose we have ClickHouse table BigDataBannerShows.LogsAggregated
We `where` как раз идут условия на всякие user_options в UserDictionary.UserInfo, status в Partners.Site и так далее
вот эти таблицы - чтобы не копировать в кликхаус их постоянно, удобно подключить как внешние словари
они раз в 5 мин обновляются и хорошо

ClickHouse doesn't ask MySQL directly on per-request basis, instead it selects all needed data every 5 minutes
and keeps it in memory.

>You don't have to keep entire dictionary in memory, there's an option to keep a limited cache, leaving full data in, for example, MySQL [Yuran aka yourock88]
>cловарь 1 раз загружается в память, тогда как при джоине вы это делаете на каждый запрос

 *Как в подзапросе сделать условие на колонку из внешней таблицы? Это вообще возможно?*
- Через словарь и getDict. [Vasilij Abrosimov]


## 5.5 Работа с джойнами

Если делать дурные запросы, то при джойнах КХ может тянуть в память одну из таблиц целиком, что может вас убить.
> дурной запрос: `ANY INNER JOIN urls2 USING id`

ограничим условие с подзапросом и `where`:

>`ANY INNER JOIN (SELECT id, url FROM urls2 WHERE id = url_id) USING id` [Tima Ber]
или
>`ANY INNER JOIN (SELECT id, count() AS shows FROM urls2 WHERE day = today) USING (id)` [Shegloff]

## collapsingMergeTree related (?)
вывести предыдущее значение строки
event - runningDifference(event) [Владимир Мюге]

## 5.6

Геннадий Алексеев, [27.02.18 14:37]
Здравствуйте. А не подскажете как составить такой запрос, у меня есть два массива во вложенном запросе, в результате я хочу получить элементы первого массива, которые не содержатся во втором?

Natalya, [27.02.18 14:44]
[In reply to Геннадий Алексеев]
как то так

Natalya, [27.02.18 14:44]
SELECT [3, 4, 8] as mas, arrayFilter(x -> x NOT IN (1,2,3,4), mas) AS res

Геннадий Алексеев, [27.02.18 14:45]
Спасибо! Попробуй

Alexey Sheglov, [27.02.18 14:48]
еще такой изврат есть:

Alexey Sheglov, [27.02.18 14:48]
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


Qq, [27.02.18 15:58]
[In reply to kamish]
не массив, а столбец преобразовать в строку
например есть столбец
А
1
2
3

нужно получить строку 1,2,3

kamish, [27.02.18 15:59]
я точно помню, что в доке есть функцич

Qq, [27.02.18 15:59]
да вот никак найти не могу

Nikolai Kochetov, [27.02.18 15:59]
можно сделать groupArray, а затем склеивать массив

Nikolai Kochetov, [27.02.18 16:01]
для массива - arrayStringConcat


# 6.  работа с кафкой и zookeeper:

## 6.1
[использование zk](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.2/bk_kafka-component-guide/content/kafka-zookeeper-multiple-apps.html) для нескольких приложений одновременно (например, кафка+clickhouse)


## 6.2 кафка->кликхауз

[как наливать данные](https://clickhouse.yandex/docs/en/table_engines/kafka.html)

tldr: при работе с кафкой надо создать две таблицы (одна типа кафака, другая, допустим, mergetree) и
1 materialised view, данные принимаются в кафку, а MV в фоне укладывает их в нужную таблицу

[использование кафки через командную строку (для тестов)]( http://cloudurable.com/blog/kafka-tutorial-kafka-from-command-line/index.html):

## 6.3 Проблемы

проблемы при укладывании данных из MV в таблицу: в логах
    StorageKafka (fromkafka): EOF reached for partition 0 offset 59651

проверить [stream_flush_interval_ms](https://clickhouse.yandex/docs/en/operations/settings/settings.html)



## 7. Проблемы и вопросы

### 7.1 Расхождения при вставке данных:
[Milovidov]

Вставка всегда 1 к 1. Если не было сообщения об ошибке - то есть, если запрос `INSERT SELECT` выполнился успешно - все данные будут вставлены.

Какие могут быть случаи, когда возникают похожие ошибки:

- вставка текстового дампа, в котором есть строки с переводом строки, экранированные как `abc\\
def`. Тогда количество строк, отображаемое wc -l, будет фактически больше реального.

- вставка в Distributed таблицу, которая развозит данные асинхронно.

- вставка в ReplicatedMergeTree полностью повторяющихся блоков данных, которые дедуплицируются;

- вставка в таблицу, которая меняет количество строк при мерже, типа Collapsing, Replacing...

- вставка в Distributed таблицу с неправильной конфигурацией кластера - например, internal_replication = 1 при использовании не-Replicated таблиц; или когда перепутали шарды и реплики;

- вставка в Replicated таблицу и чтение данных с отстающих реплик;

- вставка в Buffer таблицу и чтение из обычной таблиц



### 7.2 реплики

#### Начало репликации
Alex [@Player_a], [20.02.18 01:37]
Есть база с collapsingmergetree таблицей, размером в 100 Гб.
Появилась необходимость поднять реплику и возник вопрос - а как собственно происходит синхронизация?
Как на реплику отправить 100гигабайт, которые уже есть на мастере?

[Alexey Milovidov], [20.02.18 01:41]
Если у вас таблица уже имеет тип ReplicatedCollapsingMergeTree, то всё просто: добавление новой реплики делается запросом CREATE TABLE на новом сервере. В параметрах указывается тот же путь таблицы в ZK, но другой идентификатор реплики. После создания, реплика синхронизируется сама - идёт и скачивает все нужные данные.
Если таблица просто CollapsingMergeTree (не Replicated) - то посмотрите раздел документации "Преобразование MergeTree в ReplicatedMergeTree".

#### диагностика репликаций
В логи зукипера почти нет смысла смотреть,  лучше в кликхаусные (там будет что-то вроде pulling logs to queue).
Проще посмотреть select * from system.replicas WHERE database='db' AND table='table' FORMAT Vertical, там интересные поля last_queue_update, active_replicas, absolute_delay.
BTW Если в удобоваривом виде хочется посмотреть, что происходит внутри, то можно включить <part_log> в конфиге и смотреть в таблицу system.part_log.
[Vitaliy Lyudvichenko]


#### Exception: Could not find a column of minimum size in MergeTree
> ошибка Exception: Could not find a column of minimum size in MergeTree, part /var/lib/clickhouse//data/public/events/20180124_20180124_14437_14442_1/
> такого куска нет на этой реплике. Я сделал detach partition 201801 и attach partition 201801, при аттаче потерянный кусочек скачался с другой реплики и стало хорошо

можете на проблемной реплике вообще удалить все данные после detach из папки detached, после attach проблемная скачает всю партицию

#### Зачистка и удаление реплик

> [Vadim Metikov]
>
> Пришлось вывести сервер из кластера КХ . При заведении создаю реплицируемые таблицы и говрит, что реплика уже есть:
> Code: 253. DB::Exception: Received from localhost:9000, 127.0.0.1. DB::Exception: Replica /clickhouse/tables/01/graphite_tree/replicas/r1 already exists..
> добавлю как 4я реплика, как удалить старую r1 ?


[Max Pavlov]: Нужно удалить данные о реплике в зукипере. Это делается автоматом если на старой реплике запустить drop database или drop table, там где есть реплицируемые таблицы
Иначе нужно добавлять новую реплику с другим номером

после этого зачистить zookeepr:

`[zk: localhost:2181(CONNECTED) 9] delete /clickhouse/tables/01/graphite/replicas/r1/`
(либо через gui типа zooinspector)


#### Расхождение данных в партициях реплики
[Vladislav Denisov]
`Feb 15 16:05:59 cdn2 docker[15331]: 2018.02.15 13:05:59.045028 [ 11 ] <Debug> local.clickstream (StorageReplicatedMergeTree): Part 20180101_20180131_0_267872_24 (state Deleting) should be deleted after previous attempt before fetch`

Заметил расхождения в данных, полностью удалил партицию и перелил с зеркальной ноды через scp, теперь весь лог в таких ошибках. Можно как-то вылечить?

[Alexey Sheglov]:

1. detach partition - на обеих репликах партиция перенесется в папку detached

2. очистите папку detached на проблемной ноде

3. attach partition, и проблемная нода скачает партицию со здоровой


#### масштабирование КХ без distributed tables

один из способов без использование distributed tables:

Добавляем сервера и всё. Данные попадают в кафку, от туда мы пишем в шарды КХ, distributed таблицы на запись не используем.  Если нужно больше места или уперлись в то что памяти на объем данных сервера не хватает добавляем шард.
Соответственно после добавления шарда данные начинают размазываться немного по другому,
а т.к. старые данные переодически удаляются то и по размеру шарды постепенно выравниваются
[Kirill Shvakov]

#### Кэш КХ (его нет!)

> [Jen Zolotarev]:
> Хочу протестировать производительность Clickhouse с кастомным ключом партиционирования в сравнении с дефолтным (два одинаковых селекта в разные таблицы, но с одинаковыми данными), сейчас застрял на том, что БД кеширует данные для запроса, и поэтому результаты скорости выполнения селекта получаются смазанные.
> Как почистить этот кеш? БД перезагружал, другие запросы выполнял, но эффекта не дало.

[Kirill Shvakov]:
Там кэш ОС, у КХ нет кэша, сделайте так:

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
