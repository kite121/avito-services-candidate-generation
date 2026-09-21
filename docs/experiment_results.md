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
| M1 — Lexical retrieval | Stemmed all-field BM25 + title char-TFIDF RRF | 0.312988 | Принять lexical baseline |
| M2 — Historical retrieval | Exact context + nearest historical query, quota 20 | +0.036664 на seed 42 и +0.032682 на seed 314 к control того эксперимента | Принять historical source |
| M3 — Zero-shot dense retrieval | `multilingual-e5-large-instruct`, direct dense | 0.293394 | Оставить как complementary source |
| M6 — Hybrid fusion | M1 + M2 history 20 | 0.345071 | Контроль fusion |
| M6 — Hybrid fusion | M2 history 20 → E5 10 → M1 fill | **0.348471** | Принять текущий zero-shot pipeline |

Дополнительно M3 достиг `Recall@200 = 0.538500`; это подтверждает, что dense
source приносит в union кандидатов, которых не всегда возвращает lexical
retrieval.

## Ожидающие эксперименты

| Этап | Критерий принятия | Статус |
| --- | --- | --- |
| M4 — Lexical HPO | Подобрать параметры BM25/char-TFIDF без ухудшения Recall@50 | Ноутбук подготовлен, локальный HPO остановлен без итоговой таблицы |
| M5 — Bi-encoder fine-tuning | Fine-tuned E5 улучшает direct dense и/или M6 union | Ноутбук подготовлен; не получен валидированный результат |
| M7 — CatBoost selector | `CatBoost Recall@50 > 0.348471` на frozen proxy | Ноутбук подготовлен; запуск не завершён из-за дефицита RAM/времени |
| M8 — LLM embeddings | Instruction-tuned LLM-embedder улучшает M3/M6 | Не запускался |
| M10 — Cross-encoder reranking | Cross-encoder улучшает M6/M7 из расширенного candidate pool | Не запускался |
| M11 — Query expansion | Corpus/train-derived expansions улучшают M1/M6 без query drift | Не запускался |

## Причина незавершённых этапов

Первоначальный план предполагал больше времени на повторные Kaggle runs. Эта
оценка оказалась неверной после путаницы между общей длительностью
соревнования и фактически оставшимся до дедлайна окном. Поэтому были
приоритизированы проверенные BM25, history, dense E5 и M6 fusion, а тяжёлые
дообучение и reranking не стали заявляться без измерений.

Полная связка исходного плана, ноутбуков и итоговых статусов находится в
[experiment_roadmap.md](experiment_roadmap.md).
