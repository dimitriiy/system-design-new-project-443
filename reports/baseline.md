# Замер baseline

## Условия опыта

- время сохранения результатов: 2026-11-21 21:19:25Z
- коммит: 7260671
- ядер у docker: 14
- память docker: 8318976000 байт
- лимиты app: 500000000 nanocpus, 536870912 байт
- сценарий: load/mixed.ts
- профиль: load/profile.json
- учебный набор: 2 000 000 поездок после `make seed-large` (во время прогона сценарий создаёт дополнительные поездки)

## Итоги прогона

p(95) - 56.4 мс
dropped_iterations - 0
http_req_failed (доля ошибок)- 0%

Требование по чтению истории не выполнено, ожидается p(95) <= 50 мс.
Требование по dropped_iterations выполнено: 0 пропущенных итераций.

Анализ выполнения sql запроса (EXPLAIN (ANALYZE, BUFFERS) показывает, что для выбора 20 строк просматриваются 2 млн строк.

Тип чтения таблицы - Parallel Seq Scan
Число прочитанных строк ~ 2 млн

```sql
Limit  (cost=65608.83..65611.16 rows=20 width=104) (actual time=94.499..96.321 rows=20.00 loops=1)
   Buffers: shared hit=15925 read=38334 dirtied=1
   ->  Gather Merge  (cost=65608.83..65632.00 rows=199 width=104) (actual time=94.497..96.318 rows=20.00 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=15925 read=38334 dirtied=1
         ->  Sort  (cost=64608.80..64609.01 rows=83 width=104) (actual time=89.848..89.849 rows=15.00 loops=3)
               Sort Key: created_at DESC
               Sort Method: top-N heapsort  Memory: 30kB
               Buffers: shared hit=15925 read=38334 dirtied=1
               Worker 0:  Sort Method: top-N heapsort  Memory: 30kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 30kB
               ->  Parallel Seq Scan on trips  (cost=0.00..64606.59 rows=83 width=104) (actual time=1.110..89.630 rows=66.67 loops=3)
                     Filter: (passenger_id = 1)
                     Rows Removed by Filter: 666871
                     Buffers: shared hit=15849 read=38334 dirtied=1
 Planning:
   Buffers: shared hit=92 read=11
 Planning Time: 2.022 ms
 Execution Time: 96.530 ms
(20 rows)
```

## Вывод

Гипотеза:
Основная проблема с эффективностью SQL-запроса возникает из-за того, что для поиска записей конкретного пассажира приходится просмотреть почти всю таблицу. Это увеличивает время выполнения запроса. Операцию `Parallel Seq Scan` выполняют три процесса (`loops=3`); каждый в среднем отбрасывает 666 871 строк (`Rows Removed by Filter`).
Составной индекс по `passenger_id`, `created_at` (нужен для ORDER BY created_at DESC) сократит время выполнения запроса и уменьшит кол-во проверяемых строк.

                  Корневая страница
               [ границы диапазонов ]
                 /        |        \
                /         |         \
               ▼          ▼          ▼
        Листовая       Листовая      Листовая
        страница 1     страница 2    страница 3
       ┌───────────┐  ┌───────────┐  ┌───────────┐
       │(10, 01.11)│  │(20, 01.10)│  │(30, 01.09)│
       │(10, 05.11)│⇄ │(20, 15.11)│⇄ │(30, 21.11)│
       │(10, 20.11)│  │           │  │           │
       └───────────┘  └───────────┘  └───────────┘
             │              │              │
             ▼              ▼              ▼
                   Строки таблицы trips
