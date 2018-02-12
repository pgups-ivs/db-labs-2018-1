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

