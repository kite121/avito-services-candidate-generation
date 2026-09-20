# Результаты экспериментов

> Этот файл содержит только измеренные результаты. M7, E08 и E13 не были
> завершены до дедлайна и не являются основанием для заявления об улучшении
> финального pipeline.

## Frozen protocol

- validation: `benchmark_aligned_proxy_v1`, group-disjoint, seed 42;
- corpus: только `benchmark_items.parquet`;
- метрика: macro `Recall@50`.

## Завершённые эксперименты

| Этап | Конфигурация | Recall@50 | Решение |
| --- | --- | ---: | --- |
| M1 / E04 | Stemmed all-field BM25 + title char-TFIDF RRF | 0.312988 | Принять lexical baseline |
| M2 | Exact context + nearest historical query, quota 20 | +0.036664 на seed 42 и +0.032682 на seed 314 к control того эксперимента | Принять historical source |
| M3 / E05a | `multilingual-e5-large-instruct`, direct dense | 0.293394 | Оставить как complementary source |
| M6 / E10 | M1 + M2 history 20 | 0.345071 | Контроль fusion |
| M6 / E10 | M2 history 20 → E5 10 → M1 fill | **0.348471** | Принять текущий zero-shot pipeline |

Дополнительно M3 достиг `Recall@200 = 0.538500`; это подтверждает, что dense
source приносит в union кандидатов, которых не всегда возвращает lexical
retrieval.

## Ожидающие эксперименты

| Этап | Критерий принятия | Статус |
| --- | --- | --- |
| M4 | Подобрать параметры BM25/char-TFIDF без ухудшения Recall@50 | Ноутбук подготовлен, локальный HPO остановлен без итоговой таблицы |
| M7 / E12 | `CatBoost Recall@50 > 0.348471` на frozen proxy | Ноутбук подготовлен; запуск не завершён из-за дефицита RAM/времени |
| M5 / E08 | Fine-tuned E5 улучшает direct dense и/или M6 union | Ноутбук подготовлен; не получен валидированный результат |
| E13 | Cross-encoder улучшает M6/M7 из расширенного candidate pool | Не запускался |

## Причина незавершённых этапов

Первоначальный план предполагал больше времени на повторные Kaggle runs. Эта
оценка оказалась неверной после путаницы между общей длительностью
соревнования и фактически оставшимся до дедлайна окном. Поэтому были
приоритизированы проверенные BM25, history, dense E5 и M6 fusion, а тяжёлые
дообучение и reranking не стали заявляться без измерений.
