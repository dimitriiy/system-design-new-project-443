# Замер after-index

## Условия опыта

- дата: 2026-11-22 16:13:39Z
- коммит: 8d3a7f0
- ядер у docker: 14
- память docker: 8318976000 байт
- лимиты app: 500000000 nanocpus, 536870912 байт
- сценарий: load/mixed.ts
- профиль: load/profile.json
- учебный набор: те же 2 000 000 поездок после `make seed-large`, что и в [baseline](baseline.md) (во время прогонов сценарий создаёт дополнительные поездки)
- изменение: составной индекс `trip_passenger_id_created_at` по `passenger_id`, `created_at DESC`

## Итоги прогона

p(95) истории - 1.29 мс (до индекса - 56.4 мс)
dropped_iterations - 0
http_req_failed (доля ошибок) - 0%

Требование по чтению истории выполнено, ожидается p(95) <= 50 мс.
Требование по dropped_iterations выполнено: 0 пропущенных итераций.

Анализ выполнения того же SQL-запроса для `passenger_id = 1` (`EXPLAIN (ANALYZE, BUFFERS)`) показывает, что теперь используется индекс.

Тип чтения таблицы - Index Scan
Число строк, выданных Index Scan - 20. В базовом плане Parallel Seq Scan просматривал ~ 2 млн строк.
Отдельной сортировки в новом плане нет.

```sql
Limit  (cost=0.43..81.17 rows=20 width=104) (actual time=0.076..0.156 rows=20.00 loops=1)
  Buffers: shared hit=23 read=3
  ->  Index Scan using trip_passenger_id_created_at on trips  (cost=0.43..807.88 rows=200 width=104) (actual time=0.076..0.153 rows=20.00 loops=1)
        Index Cond: (passenger_id = 1)
        Index Searches: 1
        Buffers: shared hit=23 read=3
Planning:
  Buffers: shared hit=115 read=12 dirtied=2
Planning Time: 1.060 ms
Execution Time: 0.249 ms
(10 rows)
```

## Вывод

Составной индекс сократил работу базы: вместо просмотра всей таблицы план читает поездки одного пассажира в нужном порядке и останавливается после 20 строк. p(95) истории снизился с 56.4 до 1.29 мс и стал меньше цели 50 мс. Ошибок и пропущенных итераций нет.
