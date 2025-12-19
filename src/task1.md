# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.011..0.011 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.008..0.008 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.381 ms
   Execution Time: 0.051 ms
   ```
   
   *Объясните результат:*
   Запрос использует BRIN индекс для поиска NULL значений. План показывает Bitmap Index Scan, который эффективно находит блоки, содержащие NULL значения. В данном случае строк с NULL категорией не найдено (rows=0), но индекс позволил быстро это определить без полного сканирования таблицы.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=12.562..12.562 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1224
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.073..0.073 rows=12240 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.369 ms
   Execution Time: 12.599 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Запрос использует Bitmap Index Scan по BRIN индексу категории. BRIN индекс работает на уровне блоков, поэтому он возвращает 12240 строк (целые блоки), из которых после Recheck остается 0 строк, соответствующих условию. Затем применяется фильтр по автору, но результат все равно 0. BRIN индексы не очень эффективны для точных поисков и комбинирования условий.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=28.083..28.084 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=28.059..28.061 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.003..7.954 rows=150000 loops=1)
   Planning Time: 0.368 ms
   Execution Time: 28.162 ms
   ```
   
   *Объясните результат:*
   Запрос выполняет полное последовательное сканирование таблицы (Seq Scan), так как BRIN индекс не подходит для операций DISTINCT и ORDER BY. Планировщик выбирает HashAggregate для группировки уникальных категорий, затем сортировку. BRIN индексы эффективны для диапазонных запросов, но не для операций агрегации и сортировки.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=9.503..9.504 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=9.500..9.500 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.437 ms
   Execution Time: 9.545 ms
   ```
   
   *Объясните результат:*
   Запрос использует последовательное сканирование таблицы, так как BRIN индекс не поддерживает поиск по префиксу (LIKE 'S%'). BRIN индексы эффективны для диапазонных запросов с операторами сравнения (=, <, >, <=, >=), но не для паттерн-поиска. Планировщик правильно выбирает Seq Scan, который проверяет все 150000 строк.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=28.207..28.208 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=28.198..28.200 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.465 ms
   Execution Time: 28.250 ms
   ```
   
   *Объясните результат:*
   Несмотря на наличие индекса t_books_lower_title_idx, планировщик выбирает последовательное сканирование. Это происходит потому, что индекс по функции LOWER(title) не может быть использован для префиксного поиска (LIKE 'o%'). Для эффективного использования такого индекса нужен поиск по точному совпадению или использование операторов, которые могут использовать индекс (например, с помощью регулярных выражений или полнотекстового поиска).

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=0.862..0.863 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8818
     Heap Blocks: lossy=72
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.051..0.051 rows=720 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.350 ms
   Execution Time: 0.899 ms
   ```
   
   *Объясните результат:*
   План проверяет оба условия одновременно. По сравнению с предыдущим запросом (12.599 ms), время выполнения значительно улучшилось (0.899 ms), так как составной индекс более точно определяет релевантные блоки. Индекс вернул 720 строк (блоки), из которых после Recheck осталось 0 строк, соответствующих обоим условиям.