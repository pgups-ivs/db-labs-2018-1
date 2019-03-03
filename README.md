Лабораторные работы по курсу "Базы данных", часть 1
================================

# Расписание
Занятия в этом семестре будут в каждой из двух групп проходить один раз в две недели, начиная с 13 февраля:

Группа 1 |Группа 2 | Тема
---- | ---- | ----
13.02.2018 | 20.02.2018 | Вводное занятие. Установка PG, импорт данных, простейшие запросы. 
27.02.2018 | 06.03.2018 | Операторы DDL и DML. Разработка своей схемы, наполнение данными. 
13.03.2018 | 20.03.2018 | Выборки данных. Операторы, функции. Представления. 
27.03.2018 | 03.04.2018 | Выборки с группировками, подзапросы. Расширения стандарта SQL. 
10.04.2018 | 17.04.2018 | Функции работы со структурированными данными (JSON, XML). 
24.04.2018 | 01.05.2018 | Индексы, планы запросов, просмотр статистики. Оптимизация запросов. 
08.05.2018 | 15.05.2018 | Триггеры. Хранимые процедуры и функции. 
22.05.2018 | 29.05.2018 | Защита выполненных работ. 


# Что читать
В первую очередь стоит просмотреть обучающий раздел на сайте Postgresql.org ( [Tutorial](http://www.postgresql.org/docs/10/static/tutorial.html) ) и первые уроки на [postresqltutorial.com](http://www.postgresqltutorial.com). Также можно скачать электронные книги с shared диска в компьютерных классах.

Рекомендованный online-курс: [Погружение в СУБД (2017)](https://stepik.org/course/3203/syllabus) на Stepik.

! Upd: Дополнительно рекомендую учебник и примеры баз данных от [EDU.PostgresPro.ru](https://edu.postgrespro.ru/)

# Начало работы
Для выполнения работ необходимо [скачать](https://www.postgresql.org/download/) и установить сервер СУБД PostgreSQL. Для Windows рекомендуется дистрибутив [EnterpriseDB](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads).

Работы этого семестра будут использовать в качестве примера БД с сайта http://postgresqltutorial.com/. После установки сервера нужно скачать и импортировать образ базы данных:
в этом репозитории [dvdrental.zip](files/dvdrental.zip) или http://www.postgresqltutorial.com/postgresql-sample-database/

В zip-архив упакована резервная копия базы данных в формате tar. Инструкция по восстановлению БД с помощью утилит командной строки - в документации, [25.1 backup-dump](https://www.postgresql.org/docs/10/static/backup-dump.html#BACKUP-DUMP-RESTORE).

В получившейся базе данных должно быть 15 таблиц, 7 представлений и 8 функций:

![Restored db](files/restored_db.png)

Логическая модель загруженной базы данных:

![Диаграмма базы данных](http://www.postgresqltutorial.com/wp-content/uploads/2013/05/PostgreSQL-Sample-Database.png)

Расположение соответствующих таблицам файлов на диске можно узнать с помощью функции `pg_relation_filepath`. Пример выборки есть в документации Postgres в параграфе [Determining Disk Usage](https://www.postgresql.org/docs/10/static/disk-usage.html):

```sql
SELECT pg_relation_filepath(c.oid), n.nspname, c.relname, c.relpages, c.relkind 
FROM pg_class c JOIN pg_namespace n ON c.relnamespace = n.oid 
WHERE n.nspname = 'public'
```

> :grey_question: что обозначают символы в столбце relkind? Почему у некоторых записей в столбце path нет значения?


В столбце relpages указано количество страниц данных, записанных в соответствующий файл. Размер файла в байтах можно получить функцией `pg_total_relation_size`, а потом перевести его в более легкочитаемый вид функцией `pg_size_pretty`:

```sql
SELECT n.nspname || '.' || c.relname, c.relpages, 
  pg_size_pretty( pg_total_relation_size(c.oid) ) as "total",
  pg_size_pretty( pg_table_size(c.oid) ) as "table only",
  pg_size_pretty( pg_indexes_size(c.oid) ) as "indexes size",
  pg_size_pretty( pg_relation_size(c.oid) ) as "main size"
FROM pg_class c JOIN pg_namespace n ON c.relnamespace = n.oid 
WHERE n.nspname = 'public' AND c.relkind = 'r'
ORDER BY relpages DESC
```

> :grey_question: что возвращают функции pg_total_relation_size и pg_relation_size?

# Схемы в базе данных

Загруженная база данных доступна в __схеме__ `public` (это используемая по умолчанию область имен для объектов, создаваемых пользователями postgres, см. параграф [Schemas](https://www.postgresql.org/docs/10/static/ddl-schemas.html)). 

> :grey_question: как настраивается имя схемы, используемой по умолчанию?

Помимо неё, в базе данных обязательно присутствует системная схема, содержащая информацию о возможностях сервера и о данных, хранящихся в базе. О содержимом системной схемы можно прочитать в главе справки [Information schema](https://www.postgresql.org/docs/10/static/information-schema.html). 

Примеры запросов, получающих информацию о поддерживаемых сервером возможностях SQL:
```sql
select * from information_schema.sql_parts
select * from information_schema.sql_features WHERE is_supported = 'YES'
select * from information_schema.sql_sizing
select * from information_schema.sql_implementation_info
```
> :grey_question: какова максимальная длина названия столбца таблицы в PostgreSQL? есть ли ограничение на количество столбцов в одной таблице?

В информационной схеме доступны описания таблиц, столбцов и прочих объектов базы данных. Например, можно получить список столбцов всех таблиц базы данных:
```sql
SELECT *
FROM information_schema.columns
WHERE table_schema = 'public'
```

> :grey_question: что хранится в представлении table_privileges? в каком представлении описаны параметры функций и процедур?


# Разработка схем данных
Следующая тема для изучения - средства описания схем данных, операторы DDL - Data Definition Language. В документации Postgres есть соотвествующая глава - [Data Definition](http://www.postgresql.org/docs/10/static/ddl.html), на сайте PGT - серия уроков, начиная с [Create Table](http://www.postgresqltutorial.com/postgresql-create-table/). 

Изучаемые параграфы:
* Создание таблиц - операция [CREATE TABLE](http://www.postgresql.org/docs/10/static/sql-createtable.html)
* [Типы данных](http://www.postgresql.org/docs/10/static/datatype.html)
* [Типы ограничений](http://www.postgresql.org/docs/10/static/ddl-constraints.html)
* Создание [индексов](http://www.postgresql.org/docs/10/static/indexes.html)
* Создание [представлений](http://www.postgresql.org/docs/10/static/rules-views.html) и [схем](http://www.postgresql.org/docs/10/static/ddl-schemas.html)
* Изменение таблиц - операция [ALTER TABLE](http://www.postgresql.org/docs/10/static/ddl-alter.html)

> :grey_question: В учебной схеме следует разобрать скрипты создания таблиц и представлений, после чего доработать их так, чтобы появились следующие возможности:
  * Указать роль, которую исполнил актер в конкретном фильме
  * Описывать группу, создававшую фильм (режиссеры, сценаристы, операторы итд)
  * Сохранять описания фильмов и биографии актеров на нескольких языках
  * Указывать дату выхода фильма в прокат в разных странах
  * Сохранять в описании фильма бинарные файлы (афиши, кадры, видео-файлы)
  * Указать дату, к которой фильм должен быть возвращен из проката, в дополнение к фактической дате возврата
  * Перечислить языки озвучивания для доступной к прокату копии фильма

# Выборка данных: оператор Select
[Полный синтаксис оператора Select](http://www.postgresql.org/docs/current/static/sql-select.html)

Изучение оператора Select разделено на два занятия:

1. Базовая структура оператора select (выборки данных из одной таблицы)
  * Однострочные функции
    * [агрегатные функции](http://www.postgresql.org/docs/current/static/functions-aggregate.html) (COUNT, SUM, AVG, MAX, MIN)
    * использование [строковых](http://www.postgresql.org/docs/current/static/functions-string.html), [числовых](http://www.postgresql.org/docs/current/static/functions-math.html) и функций для работы с [датами](http://www.postgresql.org/docs/current/static/functions-datetime.html)
    * функции [преобразования типов данных](http://www.postgresql.org/docs/current/static/functions-formatting.html)
    * условные выражения: [CASE, COALESCE и NULLIF](http://www.postgresql.org/docs/current/static/functions-conditional.html)
    * [операторы сравнения строк](http://www.postgresql.org/docs/current/static/functions-matching.html): LIKE, SIMILAR TO и регулярные выражения
  * [Группировка данных](http://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-GROUP)
    * группировка данных с помощью фразы GROUP BY
    * исключение итоговых строк при помощи фразы HAVING
  * Сортировка данных - фраза [ORDER BY](http://www.postgresql.org/docs/current/static/queries-order.html)
  * Ограничение размера результата выборки - фразы [LIMIT и OFFSET](http://www.postgresql.org/docs/current/static/queries-limit.html)
2. Выборка данных из нескольких таблиц
  * [соединения таблиц](http://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-FROM)
    * декартово произведение - CROSS JOIN
    * внутренее соединение - INNER JOIN
      * эквисоединение - условия ON и USING
      * естественное соединение - условие NATURAL
    * внешние соединения (LEFT [OUTER] JOIN, RIGHT [OUTER] JOIN, FULL [OUTER] JOIN)
    * соединение более двух таблиц
    * соединение таблицы с собой
  * объединение запросов
    * [операторы UNION, INTERSECT, EXCEPT](http://www.postgresql.org/docs/current/static/queries-union.html)
  * Подзапросы
    * подзапросы в фразах FROM и HAVING
    * коррелированные подзапросы
    * использование операторов [EXISTS, ANY, SOME, ALL, IN, NOT IN](http://www.postgresql.org/docs/current/static/functions-subquery.html)

> :grey_question: В учебной схеме составьте SQL-запросы, возвращающие:
  * Названия фильмов с их категориями
  * Количество фильмов в каждой категории в порядке убывания количества фильмов
  * Количество фильмов для каждого из возрастных рейтингов
  * Количество фильмов, для которых нет трейлеров
  * Общее количество копий фильмов в каждом из магазинов
  * Количество копий каждого из фильмов в каждом магазине
  * Количество зарегистрированных в каждом из магазинов клиентов
  * email клиентов с указанием количества взятых напрокат фильмов и суммы
  * Какие 10 фильмов брали в прокат больше всего раз?
  * Какие 10 фильмов брали в прокат чаще остальных за последние 3 месяца?
  * В каком из фильмов снималось наибольшее число актеров?
  * Три самые популярные категории фильмов за последние 2 месяца
  * Сумма выручки по каждому из магазинов за последний месяц
  * Сумма выручки по всем магазинам за каждый из месяцев прошлого года
  * Сумма выручки в каждый из дней последних трех недель
