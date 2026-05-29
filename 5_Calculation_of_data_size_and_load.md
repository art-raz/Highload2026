## Размеры типов PostgreSQL / ScyllaDB

|Тип|Размер|
|---|---|
|`uuid`|16 B|
|`integer`|4 B|
|`bigint`|8 B|
|`timestamp`|8 B|
|`float`|4 B|
|`varchar(50)`|~50 B|
|`varchar(255)`|~255 B|
|`text`|переменный|

Источник размеров PostgreSQL:  
[PostgreSQL Data Types](https://www.postgresql.org/docs/current/datatype.html?utm_source=chatgpt.com)

Дополнительно каждая строка хранит:

- служебный overhead PostgreSQL/ScyllaDB
- выравнивание памяти
- metadata MVCC

Для упрощения:

```text
служебный overhead ≈ 24 B на строку
```

Это типичная оценка для PostgreSQL.

Источник:  
[PostgreSQL Storage Layout](https://www.postgresql.org/docs/current/storage-page-layout.html?utm_source=chatgpt.com)

# `media_types`

## Размер строки

Схема:

```sql
id integer
name varchar(50)
description text
```

Расчёт:

|Поле|Размер|
|---|---|
|id|4 B|
|name|~20 B|
|description|~16 B|
|overhead|24 B|

```text
4 + 20 + 16 + 24
= 64 B
```

## Нагрузка

Это маленький справочник.

Типов медиа всего несколько:

- image
- video
- gif

Запись почти отсутствует.

Допущение:

```text
<100 записей в сутки
```

Чтение:

используется при загрузке metadata media.

Допущение:

```text
~100k чтений/сутки
```

# `users`

## Размер строки

Схема:

```sql
id uuid
username varchar(50)
email varchar(100)
password_hash varchar(255)
timestamps
```

## Расчёт

|Поле|Размер|
|---|---|
|id|16 B|
|username|30 B|
|email|40 B|
|password_hash|255 B|
|created_at|8 B|
|updated_at|8 B|
|last_online|8 B|
|overhead|24 B|

Итого:

```text
16 + 30 + 40 + 255 + 8 + 8 + 8 + 24
= 389 B
```

Но PostgreSQL хранит строки с выравниванием + индексы.

Плюс:

- username index
- email index
- metadata

Берём реалистичное среднее:

```text
≈ 1 KB
```

## Запись

Изменения профиля происходят редко.

Основные операции:

- регистрация
- update profile
- last_online

Допущение:

```text
4% DAU обновляют профиль ежедневно
```

DAU:

```text
248 млн × 0.04
= 9.92 млн операций/сутки
```

QPS:

```text
9.92 млн / 86400
≈ 115 QPS
```

Peak:

```text
115 × 1.5
≈ 173
```

Округлим:

```text
≈ 200 peak
```

## Чтение

Профиль нужен:

- в ленте
- в replies
- в поиске
- в recommendations

Допущение:

```text
~4% feed requests требуют profile lookup
```

Feed read QPS:

```text
387 500
```

Тогда:

```text
387 500 × 0.04
≈ 15 500
```

Peak:

```text
≈ 22k
```

# `sessions`

## Размер строки

Схема:

```sql
id uuid
user_id uuid
token_hash varchar(255)
timestamps
```

## Расчёт

|Поле|Размер|
|---|---|
|id|16 B|
|user_id|16 B|
|token_hash|255 B|
|created_at|8 B|
|expires_at|8 B|
|overhead|24 B|

```text
16 + 16 + 255 + 8 + 8 + 24
= 327 B
```

Округляем:

```text
≈ 350 B
```

## Запись

Допущение:

каждый пользователь создаёт 1 новую session/day.

```text
248 млн / 86400
≈ 2870
```

Peak:

```text
2870 × 1.5
≈ 4300
```

## Чтение

Session lookup происходит почти на каждом API запросе.

Из раздела ingress:

```text
Peak API RPS ≈ 643k
```

Среднее:

```text
≈ 643k reads/sec
```

Peak:

```text
≈ 965k
```

# `tweets`

## Размер строки

Схема:

```sql
id uuid
author_id uuid
content text
parent_tweet_id
quoted_tweet_id
timestamps
```

## Расчёт текста

Twitter limit:

```text
280 символов
```

UTF-8:

```text
~1–2 байта/символ
```

Средний tweet text:

```text
≈ 400 B
```

Источник лимита:  
[X Character Limit Documentation](https://help.x.com/en/using-x/counting-characters?utm_source=chatgpt.com)

## Полный размер строки

|Поле|Размер|
|---|---|
|id|16|
|author_id|16|
|content|400|
|parent_tweet_id|16|
|quoted_tweet_id|16|
|timestamps|24|
|deleted_at|8|
|overhead|24|

```text
≈ 520 B
```

Но:

- TOAST
- индексы
- metadata
- JSON serialization

В разделе 2 уже считалось:

```text
≈ 1 KB
```

Используем это значение.

## Запись

Из раздела 2:

```text
465 млн tweets/day
```

QPS:

```text
465 млн / 86400
≈ 5382
```

Peak:

```text
5382 × 1.5
≈ 8073
```

## Чтение

Из раздела 2:

```text
33.48 млрд feed views/day
```

QPS:

```text
33.48 млрд / 86400
≈ 387 500
```

Peak:

```text
≈ 581k
```

# `likes`

## Размер строки

Схема:

```sql
user_id uuid
tweet_id uuid
created_at timestamp
```

## Расчёт

|Поле|Размер|
|---|---|
|user_id|16|
|tweet_id|16|
|created_at|8|
|overhead|24|

```text
16 + 16 + 8 + 24
= 64 B
```

ScyllaDB хранит compact wide rows.

Реальный размер меньше.

Используем:

```text
≈ 32–64 B
```

Берём:

```text
≈ 32 B
```

## Запись

Из раздела 2:

```text
2.8 млрд likes/day
```

QPS:

```text
2.8 млрд / 86400
≈ 32 407
```

Peak:

```text
≈ 48 610
```

## Чтение

Лайки читаются:

- вам понравилось
- counters
- рейтинг вовлеченности

Допущение:

```text
≈ 3 чтения на 1 запись
```

```text
2.8 млрд × 3
≈ 8.4 млрд/day
```

QPS:

```text
≈ 97k
```

Округлим:

```text
≈ 90k
```

# `reposts`

Полностью аналогично likes.

## Размер строки

```text
≈ 32 B
```

## Запись

Из раздела 2:

```text
320 млн/day
```

QPS:

```text
≈ 3704
```

Peak:

```text
≈ 5556
```

## Чтение

Репосты используются реже лайков.

Допущение:

```text
≈ 5 reads per repost
```

```text
320 млн × 5
≈ 1.6 млрд/day
```

# `tweet_views`

Это крупнейшая таблица системы.

## Размер строки

Схема:

```sql
user_id uuid
tweet_id uuid
viewed_at
source varchar(20)
```


## Расчёт

|Поле|Размер|
|---|---|
|user_id|16|
|tweet_id|16|
|viewed_at|8|
|source|20|
|overhead|24|

```text
16 + 16 + 8 + 20 + 24
= 84 B
```

ScyllaDB хранит компактнее.

Берём:

```text
≈ 24–48 B
```

Используем:

```text
≈ 24 B
```

## Запись

Каждый просмотр ленты → запись.

Из раздела 2:

```text
33.48 млрд/day
```

QPS:

```text
≈ 387 500
```

Peak:

```text
≈ 581k
```

## Чтение

Используется для deduplication recommendations.

Допущение:

```text
1 read на 7 просмотров
```

```text
33.48 млрд / 7
≈ 4.7 млрд/day
```

# `tweet_media`

## Размер строки

Схема содержит:

```sql
object_key
url
mime_type
size_bytes
width
height
duration
```

## Расчёт

|Поле|Размер|
|---|---|
|uuid fields|32|
|object_key|100|
|url|150|
|mime_type|20|
|metadata|40|
|timestamps|8|
|overhead|24|

```text
≈ 374 B
```

Но:

- URL хорошо сжимается
- часть metadata nullable

Среднее:

```text
≈ 128–256 B
```

Берём:

```text
≈ 128 B
```

## Запись

Медиа-посты:

```text
10.1% tweets содержат media
```

```text
465 млн × 0.101
≈ 46.9 млн/day
```

QPS:

```text
≈ 543
```

Peak:

```text
≈ 807
```

## Чтение

Медиа metadata читается почти всегда при render feed.

Допущение:

```text
≈ 30% feed items содержат media
```

```text
33.48 млрд × 0.3
≈ 10 млрд/day
```

# `tweet_counters`

## Размер строки

Схема:

```sql
likes_count bigint
reposts_count bigint
replies_count bigint
views_count bigint
```

## Расчёт

|Поле|Размер|
|---|---|
|tweet_id|16|
|4 bigint|32|
|updated_at|8|
|overhead|24|

```text
16 + 32 + 8 + 24
= 80 B
```

Округляем:

```text
≈ 64 B
```
## Запись

Каждое взаимодействие обновляет counters.

Из раздела 2:

|Тип|Обновления|
|---|---|
|likes|2.8 млрд|
|reposts|320 млн|
|tweets|465 млн|

Плюс views.

Но counters обновляются batch-ами.

Допущение:

```text
10 просмотров → 1 update
```

```text
33.48 млрд / 10
≈ 3.3 млрд updates
```

Итого:

```text
≈ 6.9 млрд/day
```

Но updates агрегируются Redis/Kafka.

В БД попадает примерно половина.

```text
≈ 3.6 млрд/day
```

QPS:

```text
≈ 41k
```
# `recommendations`

## Размер строки

Схема:

```sql
user_id
tweet_id
score
timestamp
```
## Расчёт

|Поле|Размер|
|---|---|
|user_id|16|
|tweet_id|16|
|score|4|
|timestamp|8|
|overhead|24|

```text
≈ 68 B
```

Но recommendation feed хранится пачками.

Средняя recommendation batch:

```text
~20 tweet ids
```

Redis Sorted Set overhead очень большой.

Реалистично:

```text
≈ 1–2 KB/user feed cache
```

Берём:

```text
≈ 1.5 KB
```

## Запись

Допущение:

```text
recommendation refresh 10 раз/day
```

```text
248 млн × 10
≈ 2.48 млрд updates/day
```

## Чтение

Recommendation feed читается при каждом открытии ленты.

То есть:

```text
≈ 387k read QPS
```

# `author_embeddings`

## Размер строки

Embedding:

```text
256 float32
```

```text
256 × 4
= 1024 B
```

Плюс JSON/text overhead.

```text
≈ 2 KB
```

## Запись

Пересчёт embedding:

```text
1 раз/day на автора
```

```text
24.8 млн / 86400
≈ 287
```

Peak:

```text
≈ 430
```

## Чтение

Используется recommendation retrieval.

Допущение:

```text
≈ users read QPS
≈ 15k
```

# `user_avatars`

## Размер строки

Metadata only.

|Поле|Размер|
|---|---|
|user_id|16|
|object_key|100|
|url|150|
|size_bytes|8|
|mime_type|20|
|updated_at|8|
|overhead|24|

```text
≈ 326 B
```

После сжатия metadata:

```text
≈ 128 B
```

## Запись

Совпадает с update profile:

```text
≈ 115 QPS
```

## Чтение

Аватары используются почти везде.

Но CDN снимает основную нагрузку.

Допущение:

```text
90% avatar requests уходят в CDN cache
```

До origin доходит:

```text
≈ 10%
```

```text
387k × 0.1
≈ 40k
```
