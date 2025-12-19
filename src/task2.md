## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.029..0.030 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.021..0.022 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 0.927 ms
    Execution Time: 0.065 ms
    ```
    
    *Объясните результат:*
    GIN индекс эффективно используется для полнотекстового поиска. План показывает Bitmap Index Scan по GIN индексу, который быстро находит строки, содержащие слово 'expert' в заголовке. Индекс работает с tsvector представлением текста, что позволяет выполнять быстрый поиск по лексемам. Время выполнения очень мало (0.065 ms), что демонстрирует эффективность GIN индексов для полнотекстового поиска.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.022..0.023 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.278 ms
     Execution Time: 0.050 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по первичному ключу. Поиск выполняется очень быстро (0.050 ms), так как используется B-tree индекс первичного ключа. Данные в таблице не кластеризованы, поэтому физический порядок строк может не соответствовать порядку индекса.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.055..0.055 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.370 ms
     Execution Time: 0.083 ms
     ```
     
     *Объясните результат:*
     Запрос также использует Index Scan по первичному ключу. Время выполнения немного больше (0.083 ms против 0.050 ms), но это может быть связано с особенностями выполнения. Кластеризация физически упорядочивает данные по индексу, что может улучшить производительность последовательного чтения и уменьшить количество случайных обращений к диску при работе с диапазонами данных.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.051..0.051 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.332 ms
     Execution Time: 0.077 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по индексу item_value. Поиск выполняется быстро (0.077 ms), используя B-tree индекс. Значение 'T_BOOKS' не найдено в таблице (rows=0), так как данные были заполнены как 'Value_1', 'Value_2' и т.д.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.038..0.038 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.403 ms
     Execution Time: 0.078 ms
     ```
     
     *Объясните результат:*
     Запрос также использует Index Scan по индексу item_value. Время выполнения практически идентично обычной таблице (0.078 ms против 0.077 ms). Кластеризация по первичному ключу не влияет на производительность поиска по вторичному индексу, так как данные все равно должны быть найдены через индекс и затем прочитаны из таблицы.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Производительность поиска по значению в обеих таблицах практически идентична:
     - Обычная таблица: 0.077 ms
     - Кластеризованная таблица: 0.078 ms
     
     Это объясняется тем, что кластеризация влияет на физический порядок данных по индексу кластеризации (первичному ключу), но не влияет на поиск по вторичным индексам. При поиске по item_value используется отдельный B-tree индекс, который работает одинаково в обоих случаях. Кластеризация полезна для:
     - Последовательного чтения данных по ключу кластеризации
     - Уменьшения количества случайных обращений к диску при работе с диапазонами
     - Улучшения производительности запросов, которые читают много строк в порядке кластеризации
     
     Однако для точечных запросов по вторичным индексам кластеризация не дает преимуществ.