# План экспериментов и итоговый статус

## Матрица плана

| Этап | Что планировалось проверить | Артефакт | Результат / решение |
| --- | --- | --- | --- |
| M0 — Data audit | Схемы, overlap train/corpus, labels, group-disjoint split, submission validator | [`00_m0_data_audit.ipynb`](../notebooks/00_m0_data_audit.ipynb) | `99.9894%` category match → включить category partition; `83.1%` location match → не делать location hard filter. |
| M1 — Lexical retrieval | BM25, русская нормализация, char-TFIDF и RRF | [`01_m1_lexical_retrieval.ipynb`](../notebooks/01_m1_lexical_retrieval.ipynb) | `Recall@50 = 0.312988` → включить stemmed BM25 + title char-TFIDF RRF. |
| M2 — Historical retrieval | Exact/nearest historical query → selected item без leakage | [`02_m2_historical_retrieval.ipynb`](../notebooks/02_m2_historical_retrieval.ipynb) | `+0.036664` (seed 42), `+0.032682` (seed 314) → включить nearest history, quota 20. |
| M3 — Zero-shot dense retrieval | Direct dense retrieval и semantic complementarity | [`03_m3_multilingual_e5.ipynb`](../notebooks/03_m3_multilingual_e5.ipynb) | E5: `Recall@50 = 0.293394`, `Recall@200 = 0.538500` → включить как complementary source; полный benchmark bi-encoders не завершён. |
| M4 — Lexical HPO | Подбор параметров BM25 и char-TFIDF после выбора baseline | [`04_m4_lexical_hpo.ipynb`](../notebooks/04_m4_lexical_hpo.ipynb) | Нет итоговой метрики → не менять M1 конфигурацию. |
| M5 — Bi-encoder fine-tuning | Дообучить E5 на query–positive-item и hard negatives | [`05_e08_finetune_e5.ipynb`](../notebooks/05_e08_finetune_e5.ipynb) | Нет валидированного gain → checkpoint не включать. |
| M6 — Hybrid fusion | Измерить complementarity M1, M2 и E5; выбрать quota policy | [`06_m6_hybrid_fusion.ipynb`](../notebooks/06_m6_hybrid_fusion.ipynb) | M1+M2: `0.345071`; `history 20 → E5 10 → M1 fill`: `0.348471` → принять policy на proxy. |
| M7 — CatBoost selector | Выбрать final 50 из hybrid pool по source, text и item features | [`07_m7_catboost_selector.ipynb`](../notebooks/07_m7_catboost_selector.ipynb) | Нет итоговой метрики: RAM/time limit → CatBoost не включать. |
| M8 — LLM embeddings | Проверить instruction-tuned LLM как локальный embedder и сравнить с E5 | — | Нет метрики → не включать. |
| M9 — Submission pipeline | Собрать candidates, проверить формат и создать `answer.csv` | [`09_m9_final_submission.ipynb`](../notebooks/09_m9_final_submission.ipynb) | `answer.csv` сформирован в M1 lexical fallback → dense-источник исключён из deadline run из-за нехватки времени. |
| M10 — Cross-encoder reranking | Rerank малого hybrid pool только при измеримом gain | — | Нет метрики → не включать. |
| M11 — Query expansion | Проверить corpus/train-derived query expansions без external API в inference | — | Нет метрики → не включать. |

## Запланированный shortlist моделей и методов

Этот список фиксирует именно план сравнения. Он не означает, что каждая модель
была запущена или вошла в решение.

| Направление | Планируемые модели / варианты | Что сравнить |
| --- | --- | --- |
| M3 — Zero-shot dense retrieval | `intfloat/multilingual-e5-large-instruct`, `BAAI/bge-m3`, `deepvk/USER-bge-m3`, `ai-forever/ru-en-RoSBERTa`, `Qwen/Qwen3-Embedding-0.6B` | Direct query→item Recall@K, hybrid complementarity, время encoding и RAM/VRAM на одинаковом corpus. |
| M5 — Bi-encoder fine-tuning | Лучший zero-shot bi-encoder из M3; contrastive/in-batch negatives; BM25/E5 mined weak negatives | Zero-shot против fine-tuned retrieval и влияние checkpoint на M6 fusion. |
| M8 — LLM embeddings | Локальный instruction-tuned `Qwen3-4B-Instruct-2507` как экспериментальный LLM-derived embedder | Улучшает ли контекстное LLM-представление direct dense retrieval по сравнению с E5 при приемлемом времени inference. |
| M10 — Cross-encoder reranking | Мультиязычный cross-encoder на bounded M1+M2+E5 pool | Прирост `Recall@50` относительно M6 и стоимость reranking одного запроса. |
| M11 — Query expansion | Deterministic expansion из train/corpus: синонимы, service parameters и частотные query–item terms | Gain по query slices и отсутствие query drift; external API не допускается в inference. |

## Что подтверждено измерениями

1. Лексический baseline необходим и силён для коротких сервисных запросов.
2. History добавляет кандидатов, которых не возвращает один lexical retrieval.
3. Dense E5 имеет самостоятельную semantic coverage и улучшает M6 fusion при
   ограниченной quota.
4. Жёсткое ограничение по категории сохраняет практически все положительные
   объявления и уменьшает retrieval corpus для каждого запроса.

Полная таблица численных результатов находится в
[experiment_results.md](experiment_results.md), а использованные признаки,
контроль leakage и validation protocol описаны в
[solution.md](solution.md).

## Почему часть плана не была завершена

После старта тяжёлых экспериментов стало ясно, что фактически оставшееся до
дедлайна время меньше первоначальной оценки: была путаница между общей
длительностью соревнования и доступным окном для повторных Kaggle runs.
Поэтому был выполнен минимальный набор экспериментов, достаточный для
обоснованного candidate-generation решения: M0–M3 и M6. CatBoost, fine-tuning
и cross-encoder не заявляются без полной валидации. Это ограничение по
времени, а не отрицательный вывод о самих методах.
