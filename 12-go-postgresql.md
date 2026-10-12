# Занятие 12. Работа в Go с PostgreSQL

Видео: https://www.youtube.com/watch?v=Lvi1NVyV67s (2:05:14)
Расшифровка: `transcripts/12_Lvi1NVyV67s.txt`

## Что было на занятии

| Время | Тема |
|-------|------|
| 0:01 | Общие сведения о PostgreSQL, лимиты |
| 0:02 | Индексы: B-tree по умолчанию (диапазоны, сортировка, `IS NULL`), уникальные, по выражению, `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY`, hash-индекс |
| 0:08 | Транзакции, MVCC и снапшоты. Феномены: dirty read, non-repeatable read, phantom read, аномалия сериализации. Уровни изоляции и чем PostgreSQL строже стандарта; `READ COMMITTED` по умолчанию |
| 0:22 | Партицирование и шардирование |
| 0:24 | `database/sql` и драйверы: `lib/pq`, `pgx`, `go-pg`, `bun`. DSN |
| 0:29 | Подключение: `sql.Open` не устанавливает соединение, `PingContext`; `pgx.Connect` |
| 0:33 | Пул соединений: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`; PgBouncer и режимы пулинга |
| 0:37 | Миграции `goose`: Up/Down, `go:embed`, CLI и библиотека. Как безопасно добавить `NOT NULL`-колонку; почему в проде миграции катят вручную |
| 0:45 | `*sql.DB` потокобезопасен и создаётся один раз |
| 0:48 | Фикстуры `testfixtures`: YAML → БД, защита от запуска не на тестовой базе; не подменять PostgreSQL на SQLite в тестах |
| 0:51 | Запросы: `ExecContext`, `QueryRowContext` + `Scan`, `RETURNING id`, `sql.ErrNoRows`, `QueryContext` + `rows.Next()` + `rows.Close()` |
| 1:00 | Prepared statements, выделенное соединение `Conn`, транзакции `BeginTx`/`Commit`/`Rollback` |
| 1:02 | NULL: `sql.NullInt64` и указатели |
| 1:03 | SQL-инъекции, плейсхолдеры (и где их нельзя использовать) |
| 1:04 | Неудобства `database/sql` |
| 1:20 | `sqlx`: именованные параметры, `Get`/`Select`, теги `db`, `sqlx.Named` → `sqlx.In` → `Rebind`, распаковка `args...` |
| 1:39 | `squirrel`: сборка запросов, `PlaceholderFormat(sq.Dollar)`, иммутабельность билдера, `sq.Eq`/`NotEq`/`Like`/`And`/`Or`, `ToSql()`, условные `Where`, `Join` |
| 1:58 | Тестирование: `go-txdb`, `go-sqlmock`. `EXPLAIN`, визуализаторы планов |

## Вопросы для самопроверки

1. Какие операции умеет ускорять B-tree? Когда уместен hash-индекс?
2. Что даёт `CONCURRENTLY` при создании индекса и чем за это платят?
3. Что такое MVCC?
4. Опишите четыре феномена и сопоставьте их с уровнями изоляции в PostgreSQL. Какой уровень используется по умолчанию?
5. Чем партицирование отличается от шардирования и от репликации?
6. Зачем `database/sql` нужен драйвер и как он регистрируется? Что означает `import _ "..."`?
7. В какой момент `sql.Open` обнаружит, что база недоступна?
8. Почему `*sql.DB` не открывают на каждый запрос? Какие у пула настройки?
9. Зачем PgBouncer? Какая проблема возникает с отменой запросов через него?
10. Как добавить `NOT NULL`-колонку в большую таблицу без долгой блокировки?
11. Когда использовать `Exec`, `QueryRow`, `Query`? Что вернёт `Scan`, если строки нет?
12. Почему `rows.Close()` обязателен и что такое `rows.Err()`?
13. Как получить id вставленной строки в PostgreSQL и почему не работает `LastInsertId`?
14. Как в Go представить nullable-колонку? Два способа.
15. Что такое SQL-инъекция? Почему плейсхолдером нельзя подставить имя таблицы или колонку сортировки?
16. Какие три шага нужны в `sqlx` для запроса с именованным параметром-слайсом в `IN`? Какая ошибка с `args` самая частая?
17. Почему `squirrel` — query builder, а не ORM? Почему результат вызова метода билдера нужно присваивать?
18. Чем плоха идея тестировать репозиторий на SQLite вместо PostgreSQL?
19. Зачем `EXPLAIN (ANALYZE, BUFFERS)`?

## Практика

Окружение: `docker compose` с `postgres:16`, база `shop_test`. Схема: `categories(id, name)`,
`products(id, name, category_id → categories, price, description NULL, created_at)`.

### Задание 1. Подключение и пул

1. Подключитесь через `database/sql` + `pgx/v5/stdlib`. Убедитесь, что при выключенной базе `sql.Open` не возвращает
   ошибку, а `PingContext` возвращает.
2. Настройте пул: `MaxOpenConns=5`, `MaxIdleConns=5`, `ConnMaxLifetime=5m`.
3. Запустите 50 горутин с `select pg_sleep(1)` и печатайте `db.Stats()` раз в 200 мс: `InUse`, `WaitCount`,
   `WaitDuration`. Затем задайте контексту таймаут 500 мс — что получили ожидающие?
4. Подключитесь «чистым» `pgxpool` и сравните API.

### Задание 2. Миграции

1. `goose`: миграция 1 создаёт таблицы; миграция 2 добавляет индекс по `products(category_id)`.
2. Встройте миграции через `go:embed` и запускайте из приложения по флагу `-migrate`.
3. Проверьте `up`, `down`, `status`, `redo`. Что хранится в `goose_db_version`?
4. Миграция 3: добавить `products.sku TEXT NOT NULL UNIQUE` в таблицу с миллионом строк (сгенерируйте через
   `generate_series`) без долгой блокировки: nullable-колонка → заполнение батчами → `NOT NULL` → уникальный индекс
   `CONCURRENTLY`. Почему `CREATE INDEX CONCURRENTLY` нельзя выполнять внутри транзакции и как это указать goose
   (`-- +goose NO TRANSACTION`)?

### Задание 3. Репозиторий на `database/sql`

Реализуйте `ProductRepo`: `Create(ctx, *Product) error` (заполняет ID через `RETURNING`), `GetByID` (маппит
`sql.ErrNoRows` в свой `ErrNotFound`), `List(ctx, limit, offset)`, `ListByCategoryIDs(ctx, ids []int64)`,
`Update`, `Delete` (возвращает, была ли строка — `RowsAffected`).

Требования: везде `...Context`; `defer rows.Close()` и проверка `rows.Err()`; `description` — nullable (сделайте
оба варианта: `sql.NullString` и `*string`); никакого `Sprintf` для значений.

### Задание 4. Инъекция

1. Напишите намеренно уязвимый поиск: `"... WHERE name = '" + name + "'"`. Подберите ввод, возвращающий все строки,
   и ввод, удаляющий таблицу (на тестовой базе).
2. Исправьте плейсхолдером и повторите атаку.
3. Сортировка по колонке из пользовательского ввода: плейсхолдер не работает — реализуйте белый список колонок.

### Задание 5. Транзакции и уровни изоляции

1. `Transfer(ctx, fromProduct, toProduct, qty)` (перенос остатка) в транзакции: `defer tx.Rollback()` + `Commit`.
   Напишите хелпер `WithTx(ctx, db, func(tx *sql.Tx) error) error`.
2. Воспроизведите в двух сессиях `psql` или двумя горутинами:
   - non-repeatable read на `READ COMMITTED`;
   - его отсутствие на `REPEATABLE READ`;
   - lost update (обе транзакции читают остаток и пишут `остаток − 1`) и три способа лечения: `SELECT ... FOR UPDATE`,
     атомарный `UPDATE ... SET qty = qty - 1`, уровень `SERIALIZABLE` с повтором при ошибке `40001`.
3. Напишите обёртку с ретраями для ошибок сериализации.

### Задание 6. sqlx

Перепишите репозиторий: структуры с тегами `db`, `GetContext`/`SelectContext`, `NamedExecContext` для вставки из
структуры, запрос с `IN` через `sqlx.Named` → `sqlx.In` → `db.Rebind`. Намеренно передайте `args` без `...` и
прочитайте ошибку. Сравните объём кода с заданием 3.

### Задание 7. squirrel

Метод `Search(ctx, filter ProductFilter)`, где любой параметр опционален: `CategoryIDs []int64`, `NameLike string`,
`MinPrice, MaxPrice *int64`, `HasDescription *bool`, `SortBy`, `Limit`, `Offset`.

1. Соберите запрос через `squirrel` с `sq.Dollar`; выполните через `sqlx`.
2. Табличный unit-тест на `ToSql()` для комбинаций фильтров (база не нужна).
3. Проверьте иммутабельность: вызовите `.Where(...)` без присваивания.
4. Сначала напишите ту же функцию руками на `strings.Builder` — чтобы почувствовать разницу.

### Задание 8. Тесты репозитория

1. Интеграционные тесты на реальном PostgreSQL (docker-compose или `testcontainers-go`), build-тег `integration`.
2. Данные — через `testfixtures`; база обязана содержать `test` в имени.
3. Изоляция тестов: `go-txdb` или транзакция с откатом в каждом тесте.
4. Один тест сервиса — на `go-sqlmock`. Запишите минусы подхода: что такой тест не поймает?

### Задание 9. EXPLAIN

1. Миллион продуктов. `EXPLAIN (ANALYZE, BUFFERS)` для `WHERE category_id = 5`, `WHERE lower(name) = 'x'`,
   `ORDER BY created_at DESC LIMIT 20`, `LIMIT 20 OFFSET 900000` — до и после подходящих индексов.
2. Визуализируйте один план на explain.dalibo.com (`FORMAT JSON`).
3. Замените пагинацию через `OFFSET` на keyset (`WHERE id > $1 ORDER BY id LIMIT $2`) и сравните планы.

### Задание 10. Связь с проектом

Замените in-memory хранилище в `product-service` (занятие 11) на PostgreSQL: сервис не меняется, меняется только
реализация интерфейса `Repository`. Миграции — в отдельном каталоге; интеграционные тесты репозитория; unit-тесты
сервиса остаются на моках.

## Критерии готовности

- Ни одного запроса без контекста, без закрытия `rows` и без проверки ошибок.
- Можете воспроизвести и вылечить lost update тремя способами.
- Динамический поиск собирается без конкатенации строк и покрыт unit-тестами на SQL.
- Понимаете, что выбрать для задачи: `database/sql`, `sqlx`, `squirrel`, `pgx` напрямую.
