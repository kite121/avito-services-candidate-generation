# Avito Services Candidate Generation

Репозиторий решения для задачи candidate generation в поиске услуг Авито.
Для каждого поискового запроса pipeline возвращает до 50 `item_id`; основная
метрика — macro `Recall@50`.

Главный файл для воспроизведения полного гибридного pipeline —
[notebooks/09_m9_final_submission.ipynb](notebooks/09_m9_final_submission.ipynb).
Он запускается в Kaggle, строит кандидатов и валидирует формат `answer.csv`.

## Быстрый запуск полного pipeline

1. Создайте Kaggle Notebook из `notebooks/09_m9_final_submission.ipynb`.
2. Подключите Kaggle Dataset с `train.parquet`, `benchmark_queries.parquet` и
   `benchmark_items.parquet`.
3. Включите Internet и GPU. Добавьте Kaggle Secrets
   `CLEARML_API_ACCESS_KEY` и `CLEARML_API_SECRET_KEY`; host-параметры ClearML
   подключаются опционально.
4. Выполните **Run All**.
5. Скачайте единственный submission-файл `answer.csv` из output-каталога,
   путь к которому напечатает финальная ячейка.

Вычисление выполняется локально в Kaggle на открытых весах
`intfloat/multilingual-e5-large-instruct`; внешние inference API не
используются.

## Прозрачный финальный статус

В репозитории сохранены все подготовленные и измеренные этапы. На frozen
validation proxy подтверждены lexical BM25 + char-TFIDF, historical retrieval,
zero-shot dense E5 и их M6 fusion. M9 сформирован как финальный Kaggle
submission workflow и завершён в M1 lexical fallback: dense-источник был
исключён из deadline run из-за нехватки времени. CatBoost, fine-tuning и
cross-encoder не получили проверенных результатов, поэтому эти компоненты не
следует представлять как финально подтверждённые улучшения.

Причина — ошибка планирования: первоначальная оценка оставшегося времени была
сделана исходя из более длинного окна работ после путаницы между общей
длительностью соревнования и фактически оставшимся временем. В
[docs/experiment_results.md](docs/experiment_results.md) разделены завершённые
измерения и подготовленные, но не завершённые эксперименты.

## Репозиторий

```text
data/        Инструкция по локальному размещению исходных данных
docs/        Описание решения и результаты экспериментов
notebooks/   Воспроизводимые этапы M0–M9
```

Данные, embeddings, model checkpoints, ClearML credentials и итоговые CSV
намеренно не хранятся в Git. Полное описание подхода — в
[docs/solution.md](docs/solution.md); измеренные результаты — в
[docs/experiment_results.md](docs/experiment_results.md). Исходный план,
гипотезы и финальный статус каждой ветки — в
[docs/experiment_roadmap.md](docs/experiment_roadmap.md).

## Ноутбуки

| Файл | Назначение |
| --- | --- |
| `00_m0_data_audit.ipynb` | Аудит данных и frozen validation protocol |
| `01_m1_lexical_retrieval.ipynb` | BM25, char-TFIDF и русская нормализация |
| `02_m2_historical_retrieval.ipynb` | Historical query retrieval без leakage |
| `03_m3_multilingual_e5.ipynb` | Direct dense retrieval с multilingual-E5 |
| `04_m4_lexical_hpo.ipynb` | Подготовленный поиск параметров BM25 и char-TFIDF |
| `05_e08_finetune_e5.ipynb` | Time-boxed fine-tuning E5; запускается только при наличии времени |
| `06_m6_hybrid_fusion.ipynb` | Измерение complementarity и quota fusion |
| `07_m7_catboost_selector.ipynb` | Leakage-safe CatBoost selector |
| `09_m9_final_submission.ipynb` | Финальная генерация и validation `answer.csv` |
