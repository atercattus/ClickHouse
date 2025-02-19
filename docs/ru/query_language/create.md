## CREATE DATABASE {#query_language-create-database}

Создает базу данных.

```sql
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster] [ENGINE = engine(...)]
```

### Секции

- `IF NOT EXISTS`

    Если база данных с именем `db_name` уже существует, то ClickHouse не создаёт базу данных и:
    - Не генерирует исключение, если секция указана.
    - Генерирует исключение, если секция не указана.

- `ON CLUSTER`

    ClickHouse создаёт базу данных `db_name` на всех серверах указанного кластера.

- `ENGINE`

    - [MySQL](../database_engines/mysql.md)

        Позволяет получать данные с удаленного сервера MySQL.

    По умолчанию ClickHouse использует собственный [движок баз данных](../database_engines/index.md).

## CREATE TABLE {#create-table-query}

Запрос `CREATE TABLE` может иметь несколько форм.

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [compression_codec] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [compression_codec] [TTL expr2],
    ...
) ENGINE = engine
```

Создаёт таблицу с именем name в БД db или текущей БД, если db не указана, со структурой, указанной в скобках, и движком engine.
Структура таблицы представляет список описаний столбцов. Индексы, если поддерживаются движком, указываются в качестве параметров для движка таблицы.

Описание столбца, это `name type`, в простейшем случае. Пример: `RegionID UInt32`.
Также могут быть указаны выражения для значений по умолчанию - смотрите ниже.

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name AS [db2.]name2 [ENGINE = engine]
```

Создаёт таблицу с такой же структурой, как другая таблица. Можно указать другой движок для таблицы. Если движок не указан, то будет выбран такой же движок, как у таблицы `db2.name2`.

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name AS table_fucntion()
```

Создаёт таблицу с такой же структурой и данными, как результат соответствующей табличной функцией.

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name ENGINE = engine AS SELECT ...
```

Создаёт таблицу со структурой, как результат запроса `SELECT`, с движком engine, и заполняет её данными из SELECT-а.

Во всех случаях, если указано `IF NOT EXISTS`, то запрос не будет возвращать ошибку, если таблица уже существует. В этом случае, запрос будет ничего не делать.

После секции `ENGINE` в запросе могут использоваться и другие секции в зависимости от движка. Подробную документацию по созданию таблиц смотрите в описаниях [движков таблиц](../operations/table_engines/index.md#table_engines).

### Значения по умолчанию {#create-default-values}

В описании столбца, может быть указано выражение для значения по умолчанию, одного из следующих видов:
`DEFAULT expr`, `MATERIALIZED expr`, `ALIAS expr`.
Пример: `URLDomain String DEFAULT domain(URL)`.

Если выражение для значения по умолчанию не указано, то в качестве значений по умолчанию будут использоваться нули для чисел, пустые строки для строк, пустые массивы для массивов, а также `0000-00-00` для дат и `0000-00-00 00:00:00` для дат с временем. NULL-ы не поддерживаются.

В случае, если указано выражение по умолчанию, то указание типа столбца не обязательно. При отсутствии явно указанного типа, будет использован тип выражения по умолчанию. Пример: `EventDate DEFAULT toDate(EventTime)` - для столбца EventDate будет использован тип Date.

При наличии явно указанного типа данных и выражения по умолчанию, это выражение будет приводиться к указанному типу с использованием функций приведения типа. Пример: `Hits UInt32 DEFAULT 0` - имеет такой же смысл, как `Hits UInt32 DEFAULT toUInt32(0)`.

В качестве выражения для умолчания, может быть указано произвольное выражение от констант и столбцов таблицы. При создании и изменении структуры таблицы, проверяется, что выражения не содержат циклов. При INSERT-е проверяется разрешимость выражений - что все столбцы, из которых их можно вычислить, переданы.

`DEFAULT expr`

Обычное значение по умолчанию. Если в запросе INSERT не указан соответствующий столбец, то он будет заполнен путём вычисления соответствующего выражения.

`MATERIALIZED expr`

Материализованное выражение. Такой столбец не может быть указан при INSERT, то есть, он всегда вычисляется.
При INSERT без указания списка столбцов, такие столбцы не рассматриваются.
Также этот столбец не подставляется при использовании звёздочки в запросе SELECT. Это необходимо, чтобы сохранить инвариант, что дамп, полученный путём `SELECT *`, можно вставить обратно в таблицу INSERT-ом без указания списка столбцов.

`ALIAS expr`

Синоним. Такой столбец вообще не хранится в таблице.
Его значения не могут быть вставлены в таблицу, он не подставляется при использовании звёздочки в запросе SELECT.
Он может быть использован в SELECT-ах - в таком случае, во время разбора запроса, алиас раскрывается.

При добавлении новых столбцов с помощью запроса ALTER, старые данные для этих столбцов не записываются. Вместо этого, при чтении старых данных, для которых отсутствуют значения новых столбцов, выполняется вычисление выражений по умолчанию на лету. При этом, если выполнение выражения требует использования других столбцов, не указанных в запросе, то эти столбцы будут дополнительно прочитаны, но только для тех блоков данных, для которых это необходимо.

Если добавить в таблицу новый столбец, а через некоторое время изменить его выражение по умолчанию, то используемые значения для старых данных (для данных, где значения не хранились на диске) поменяются. Также заметим, что при выполнении фоновых слияний, данные для столбцов, отсутствующих в одном из сливаемых кусков, записываются в объединённый кусок.

Отсутствует возможность задать значения по умолчанию для элементов вложенных структур данных.

### Ограничения (constraints) {#constraints}

Наряду с объявлением столбцов можно объявить ограничения на значения в столбцах таблицы:

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [compression_codec] [TTL expr1],
    ...
    CONSTRAINT constraint_name_1 CHECK boolean_expr_1,
    ...
) ENGINE = engine
```

`boolean_expr_1` может быть любым булевым выражением, состоящим из операторов сравнения или функций. При наличии одного или нескольких ограничений в момент вставки данных выражения ограничений будут проверяться на истинность для каждой вставляемой строки данных. В случае, если в теле INSERT запроса придут некорректные данные — клиент получит исключение с описанием нарушенного ограничения.

Добавление большого числа ограничений может негативно повлиять на производительность `INSERT` запросов.

### Выражение для TTL

Определяет время хранения значений. Может быть указано только для таблиц семейства MergeTree. Подробнее смотрите в [TTL для столбцов и таблиц](../operations/table_engines/mergetree.md#table_engine-mergetree-ttl).

### Кодеки сжатия столбцов

По умолчанию, ClickHouse применяет к столбцу метод сжатия, определённый в [конфигурации сервера](../operations/server_settings/settings.md#compression). Кроме этого, можно задать метод сжатия для каждого отдельного столбца в запросе `CREATE TABLE`.

```sql
CREATE TABLE codec_example
(
    dt Date CODEC(ZSTD),
    ts DateTime CODEC(LZ4HC),
    float_value Float32 CODEC(NONE),
    double_value Float64 CODEC(LZ4HC(9))
    value Float32 CODEC(Delta, ZSTD)
)
ENGINE = <Engine>
...
```

Если задать кодек для столбца, то кодек по умолчанию не применяется. Кодеки можно последовательно комбинировать, например, `CODEC(Delta, ZSTD)`. Чтобы выбрать наиболее подходящую для вашего проекта комбинацию кодеков, необходимо провести сравнительные тесты, подобные тем, что описаны в статье Altinity [New Encodings to Improve ClickHouse Efficiency](https://www.altinity.com/blog/2019/7/new-encodings-to-improve-clickhouse).

!!!warning "Предупреждение"
    Нельзя распаковать базу данных ClickHouse с помощью сторонних утилит наподобие `lz4`. Необходимо использовать специальную утилиту [clickhouse-compressor](https://github.com/yandex/ClickHouse/tree/master/dbms/programs/compressor).

Сжатие поддерживается для следующих движков таблиц:

- [MergeTree family](../operations/table_engines/mergetree.md)
- [Log family](../operations/table_engines/log_family.md)
- [Set](../operations/table_engines/set.md)
- [Join](../operations/table_engines/join.md)

ClickHouse поддерживает кодеки общего назначения и специализированные кодеки.

#### Специализированные кодеки {#create-query-specialized-codecs}

Эти кодеки разработаны для того, чтобы, используя особенности данных сделать сжатие более эффективным. Некоторые из этих кодеков не сжимают данные самостоятельно. Они готовят данные для кодеков общего назначения, которые сжимают подготовленные данные эффективнее, чем неподготовленные.

Специализированные кодеки:

- `Delta(delta_bytes)` — Метод, в котором исходные значения заменяются разностью двух соседних значений, за исключением первого значения, которое остаётся неизменным. Для хранения разниц используется до `delta_bytes`, т.е. `delta_bytes` — это максимальный размер исходных данных. Возможные значения `delta_bytes`: 1, 2, 4, 8. Значение по умолчанию для `delta_bytes` равно `sizeof(type)`, если результат 1, 2, 4, or 8. Во всех других случаях  — 1.
- `DoubleDelta` — Вычисляется разницу от разниц и сохраняет её в компакном бинарном виде. Оптимальная степень сжатия достигается для монотонных последовательностей с постоянным шагом, наподобие временных рядов. Можно использовать с любым типом данных фиксированного размера. Реализует алгоритм, используемый в TSDB Gorilla, поддерживает 64-битные типы данных. Использует 1 дополнительный бит для 32-байтовых значений: 5-битные префиксы вместо 4-битных префиксов. Подробнее читайте в разделе "Compressing Time Stamps" документа [Gorilla: A Fast, Scalable, In-Memory Time Series Database](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf).
- `Gorilla` — Вычисляет XOR между текущим и предыдущим значением и записывает результат в компактной бинарной форме. Еффективно сохраняет ряды медленно изменяющихся чисел с плавающей запятой, поскольку наилучший коэффициен сжатия достигается, если соседние значения одинаковые. Реализует алгоритм, используемый в TSDB Gorilla, адаптируя его для работы с 64-битными значениями. Подробнее читайте в разделе "Compressing Values" документа [Gorilla: A Fast, Scalable, In-Memory Time Series Database](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf).
- `T64` — Метод сжатия который обрезает неиспользуемые старшие биты целочисленных значений (включая `Enum`, `Date` и `DateTime`). На каждом шаге алгоритма, кодек помещает блок из 64 значений в матрицу 64✕64, транспонирует её, обрезает неиспользуемые биты, а то, что осталось возвращает в виде последовательности. Неиспользуемые биты, это биты, которые не изменяются от минимального к максимальному на всём диапазоне значений куска данных.

Кодеки `DoubleDelta` и `Gorilla` используются в TSDB Gorilla как компоненты алгоритма сжатия. Подход Gorilla эффективен в сценариях, когда данные представляют собой медленно изменяющиеся во времени величины. Метки времени эффективно сжимаются кодеком `DoubleDelta`, а значения кодеком `Gorilla`. Например, чтобы создать эффективно хранящуюся таблицу, используйте следующую конфигурацию:

```sql
CREATE TABLE codec_example
(
    timestamp DateTime CODEC(DoubleDelta),
    slow_values Float32 CODEC(Gorilla)
)
ENGINE = MergeTree()
```

#### Кодеки общего назначения {#create-query-common-purpose-codecs}

Кодеки:

- `NONE` — без сжатия.
- `LZ4` — [алгоритм сжатия без потерь](https://github.com/lz4/lz4) используемый по умолчанию. Применяет быстрое сжатие LZ4.
- `LZ4HC[(level)]` — алгоритм LZ4 HC (high compression) с настраиваемым уровнем сжатия. Уровень по умолчанию — 9. Настройка `level <= 0` устанавливает уровень сжания по умолчанию. Возможные уровни сжатия: [1, 12]. Рекомендуемый диапазон уровней: [4, 9].
- `ZSTD[(level)]` — [алгоритм сжатия ZSTD](https://en.wikipedia.org/wiki/Zstandard) с настраиваемым уровнем сжатия `level`. Возможные уровни сжатия: [1, 22]. Уровень сжатия по умолчанию: 1.

Высокие уровни сжатия полезны для ассимметричных сценариев, подобных "один раз сжал, много раз распаковал". Высокие уровни сжатия подразумеваю лучшее сжатие, но большее использование CPU.


## Временные таблицы

ClickHouse поддерживает временные таблицы со следующими характеристиками:

- временные таблицы исчезают после завершения сессии; в том числе, при обрыве соединения;
- Временная таблица использует только модуль памяти.
- Невозможно указать базу данных для временной таблицы. Временные таблицы создается вне баз данных.
- если временная таблица имеет то же имя, что и некоторая другая, то, при упоминании в запросе без указания БД, будет использована временная таблица;
- при распределённой обработке запроса, используемые в запросе временные таблицы, передаются на удалённые серверы.

Чтобы создать временную таблицу, используйте следующий синтаксис:

```sql
CREATE TEMPORARY TABLE [IF NOT EXISTS] table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
)
```

В большинстве случаев, временные таблицы создаются не вручную, а при использовании внешних данных для запроса, или при распределённом `(GLOBAL) IN`. Подробнее см. соответствующие разделы

## Распределенные DDL запросы (секция ON CLUSTER)

Запросы `CREATE`, `DROP`, `ALTER`, `RENAME` поддерживают возможность распределенного выполнения на кластере.
Например, следующий запрос создает распределенную (Distributed) таблицу `all_hits` на каждом хосте в `cluster`:

```sql
CREATE TABLE IF NOT EXISTS all_hits ON CLUSTER cluster (p Date, i Int32) ENGINE = Distributed(cluster, default, hits)
```

Для корректного выполнения таких запросов необходимо на каждом хосте иметь одинаковое определение кластера (для упрощения синхронизации конфигов можете использовать подстановки из ZooKeeper). Также необходимо подключение к ZooKeeper серверам.
Локальная версия запроса в конечном итоге будет выполнена на каждом хосте кластера, даже если некоторые хосты в данный момент не доступны. Гарантируется упорядоченность выполнения запросов в рамках одного хоста. Для реплицированных таблиц не поддерживаются запросы `ALTER`.

## CREATE VIEW

```sql
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...
```

Создаёт представление. Представления бывают двух видов - обычные и материализованные (MATERIALIZED).

Обычные представления не хранят никаких данных, а всего лишь производят чтение из другой таблицы. То есть, обычное представление - не более чем сохранённый запрос. При чтении из представления, этот сохранённый запрос, используется в качестве подзапроса в секции FROM.

Для примера, пусть вы создали представление:

```sql
CREATE VIEW view AS SELECT ...
```

и написали запрос:

```sql
SELECT a, b, c FROM view
```

Этот запрос полностью эквивалентен использованию подзапроса:

```sql
SELECT a, b, c FROM (SELECT ...)
```

Материализованные (MATERIALIZED) представления хранят данные, преобразованные соответствующим запросом SELECT.

При создании материализованного представления, нужно обязательно указать ENGINE - движок таблицы для хранения данных.

Материализованное представление устроено следующим образом: при вставке данных в таблицу, указанную в SELECT-е, кусок вставляемых данных преобразуется этим запросом SELECT, и полученный результат вставляется в представление.

Если указано POPULATE, то при создании представления, в него будут вставлены имеющиеся данные таблицы, как если бы был сделан запрос `CREATE TABLE ... AS SELECT ...` . Иначе, представление будет содержать только данные, вставляемые в таблицу после создания представления. Не рекомендуется использовать POPULATE, так как вставляемые в таблицу данные во время создания представления, не попадут в него.

Запрос `SELECT` может содержать `DISTINCT`, `GROUP BY`, `ORDER BY`, `LIMIT`... Следует иметь ввиду, что соответствующие преобразования будут выполняться независимо, на каждый блок вставляемых данных. Например, при наличии `GROUP BY`, данные будут агрегироваться при вставке, но только в рамках одной пачки вставляемых данных. Далее, данные не будут доагрегированы. Исключение - использование ENGINE, производящего агрегацию данных самостоятельно, например, `SummingMergeTree`.

Недоработано выполнение запросов `ALTER` над материализованными представлениями, поэтому они могут быть неудобными для использования. Если материализованное представление использует конструкцию ``TO [db.]name``, то можно выполнить ``DETACH`` представления, ``ALTER`` для целевой таблицы и последующий ``ATTACH`` ранее отсоединенного (``DETACH``) представления.

Представления выглядят так же, как обычные таблицы. Например, они перечисляются в результате запроса `SHOW TABLES`.

Отсутствует отдельный запрос для удаления представлений. Чтобы удалить представление, следует использовать `DROP TABLE`.

[Оригинальная статья](https://clickhouse.yandex/docs/ru/query_language/create/) <!--hide-->
