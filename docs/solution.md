# Описание решения

> Статус на момент дедлайна: завершены и измерены M1, M2, M3 и M6. M7,
> fine-tuning E5 и cross-encoder подготовлены, но не получили итоговой
> верификации; они не считаются частью подтверждённой конфигурации.

## 1. Постановка

Для каждого `query_id` необходимо выбрать не более 50 объявлений из
`benchmark_items.parquet`. Качество измеряется macro `Recall@50`: порядок
внутри ответа на метрику не влияет, но пропущенное на retrieval-этапе
объявление уже не может вернуть последующий ранжировщик.

Итоговый запуск находится в `notebooks/09_m9_final_submission.ipynb` и создаёт
CSV строго с колонками `query_id,answer`.

## 2. Данные и признаки

Используются только предоставленные таблицы `train.parquet`,
`benchmark_queries.parquet` и `benchmark_items.parquet`.

- `search_query` — главный текстовый сигнал для BM25 и title char-TFIDF.
- `search_infm_params_text` добавляется к dense query text и участвует в
  идентификации полного historical query context.
- `search_category` задаёт жёсткий retrieval partition:
  `item_category_id == search_category`. При отсутствии такой категории
  используется fallback на весь корпус.
- `item_title_raw`, `item_infm_params_text` и `item_description_raw` — тексты
  объявлений. Для dense encoder описание ограничивается первыми 1 500
  символами.
- `item_location_id`, цена, рейтинг, число отзывов и контактные флаги не
  используются как жёсткие retrieval-фильтры. Они будут входами CatBoost
  только если M7 подтвердит прирост на frozen proxy.

## 3. Кандидатогенерация

Проверенный на frozen proxy zero-shot pipeline состоит из трёх источников.

1. **M1 lexical.** Stemmed Russian all-field BM25 и title char-TFIDF
   объединяются Reciprocal Rank Fusion.
2. **M2 history.** Для полного search context используются клики из train;
   для похожих query применяется char-TFIDF поиск по historical query texts.
   В validation history не содержит целевых query groups.
3. **M3 dense.** `intfloat/multilingual-e5-large-instruct` кодирует query и
   item texts, нормализует embeddings и выполняет точный inner-product search
   внутри категории.

Измеренная M6 policy: сначала до 20 historical candidates, затем до 10 dense
candidates, затем lexical RRF до лимита 50. Все ID дедуплицируются. Она дала
лучший измеренный proxy score среди завершённых конфигураций.

M7 CatBoost и E08 fine-tuning являются условными улучшениями: они могли бы
войти в итоговую конфигурацию только при измеренном положительном gain
относительно M6. Такой gain до дедлайна не был подтверждён. E13 cross-encoder
также не запускался.

## 4. Проверка качества до submission

Используется frozen `benchmark_aligned_proxy_v1`: из train оставляются
позитивы, чьи `item_id` присутствуют в benchmark corpus; затем query groups
разделяются group-disjoint split с seed 42. Метрика — собственная реализация
macro `Recall@50`.

Перед сохранением submission M9 проверяет:

- ровно одну строку на каждый benchmark `query_id`;
- только колонки `query_id` и `answer`;
- не более 50 уникальных `item_id` в строке;
- существование каждого ID в benchmark corpus;
- формат `item_id` из 16 lowercase hex-символов.

Таблица измеренных метрик находится в
[experiment_results.md](experiment_results.md).

## 5. Анализ ошибок и принятые решения

- **Неверная категория.** Совпадение категории положительного item с
  `search_category` составляет 99.9894%, поэтому category partition даёт
  большое ускорение без существенной потери полноты.
- **Локация.** Она совпадает только в 83.1% train positives. Поэтому локация
  не стала жёстким фильтром: иначе Recall искусственно ограничивался бы ещё
  до retrieval.
- **Лексические вариации.** Короткие запросы, морфология и разные формулировки
  услуг приводят к lexical misses; их частично закрывают Russian stemming,
  character n-grams и dense E5.
- **Leakage из history.** Исторические кандидаты строятся только из разрешённой
  train-части. Для M7 применяются OOF folds, поэтому target group не может
  передать в свои признаки собственный positive click.
- **Неполные implicit labels.** Нерелевантные retrieved candidates не
  интерпретируются как абсолютная истина. В M7 они используются как weak
  negatives только внутри candidate pool, а решение принимается по Recall@50
  на unseen groups.

## 6. Воспроизведение

1. Откройте `notebooks/09_m9_final_submission.ipynb` в Kaggle.
2. Подключите три исходных Parquet как Kaggle Input, включите Internet и GPU.
3. Добавьте live ClearML secrets `CLEARML_API_ACCESS_KEY` и
   `CLEARML_API_SECRET_KEY`.
4. Выполните ноутбук сверху вниз.
5. Скачайте созданный `answer.csv` из `/kaggle/working/.../answer.csv`.

Внутри M9 используется открытая модель `intfloat/multilingual-e5-large-instruct`
и библиотеки `sentence-transformers`, `scikit-learn`, `snowballstemmer` и
`pandas`. Inference выполняется в самом Kaggle runtime.

## 7. Ограничения финального состояния

Полный M9 notebook был подготовлен как воспроизводимый Kaggle pipeline, но его
новый end-to-end запуск с dense encoding не был завершён в оставшееся до
дедлайна время. Это не является отрицательным результатом E5: M3 и M6 уже
измерили его вклад на frozen proxy. Но это означает, что CatBoost, fine-tuning,
cross-encoder и новый full-pipeline submission нельзя заявлять как
подтверждённые результаты.

Причина — ошибка в оценке времени: план работ исходил из более длинного окна
после путаницы между общей продолжительностью соревнования и реально
оставшимся временем. Все незавершённые пункты сохранены как воспроизводимые
ноутбуки, но отмечены в результатах именно как незавершённые.
