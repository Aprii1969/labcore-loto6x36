```markdown
# LABCORE Checkpoint v1

Этот контрольный снимок фиксирует работоспособную версию всей методологии: от извлечения истории до генерации комбинаций.

---

## Содержание репозитория

```
LABCORE
├── checkpoint_v1
│   ├── config
│   │   └── config.yml
│   ├── data
│   │   ├── labcore_draws.csv       # исходные тиражи (все столбцы)
│   │   ├── main_history.csv        # история основных шаров (draw, num1…num6, mode)
│   │   └── bonus_history.csv       # история бонусных шаров
│   ├── features
│   │   ├── recency.py              # add_recency()
│   │   ├── frequencies.py          # add_sliding_freq()
│   │   ├── periodicity.py          # add_periodicity()
│   │   ├── mode_freq.py            # add_mode_specific_freq()
│   │   ├── structure.py            # add_structural()
│   │   ├── correlations.py         # add_correlations()
│   │   ├── markov.py               # add_markov_transitions()
│   │   └── mode_ohe.py             # add_mode_ohe()
│   ├── utils
│   │   └── structure_quota.py      # compute_structure_quota()
│   ├── generate_structured.py      # generate_structured_combinations()
│   ├── extract_main_history.py     # извлечение основных шаров в main_history.csv
│   ├── extract_bonus_history.py    # извлечение бонусных шаров в bonus_history.csv
│   ├── training_main_ensemble.py   # обучение RF→LR ансамбля для основных шаров
│   ├── training_bonus_ensemble.py  # обучение ансамбля для бонусных шаров
│   ├── ai_controller.py            # orchestration pipeline
│   ├── requirements.txt            # все зависимости
│   └── README.md                   # этот файл
└── models                          # после обучения
    ├── main_rf.pkl
    ├── main_meta.pkl
    ├── bonus_rf.pkl
    └── bonus_meta.pkl
```

---

## Описание ключевых файлов

### config/config.yml  
Основной конфиг, настройки по режимам day/evening:

```yaml
data_path: data/labcore_draws.csv

modes:
  day:
    main:
      window_short: 20
      window_long: 1000
      corr_window:  100
      test_size:    0.2
      random_state: 42
      ensemble:
        rf_estimators: 200
        rf_model:      models/main_rf.pkl
        meta_model:    models/main_meta.pkl
      top_k:        6

    bonus:
      window_freq:  20
      seq_len:      10
      test_size:    0.2
      random_state: 42
      ensemble:
        rf_model:    models/bonus_rf.pkl
        meta_model:  models/bonus_meta.pkl
      top_k:        3
```

- **window_short/long** – длины окон для скользящих частот.
- **corr_window** – размер окна для подсчёта пар/троек.
- **ensemble** – пути и параметры RF и meta LR.
- **top_k** – K в Hits@K.

---

### data/  
- **labcore_draws.csv** – полный исторический дамп тиражей (с исходными столбцами «Розіграш», «Кулька номер i», «Дата проведення»).  
- **main_history.csv** – результат `extract_main_history.py`: столбцы `draw`, `num1`…`num6`, `mode`.  
- **bonus_history.csv** – результат `extract_bonus_history.py`.

---

### extract_*.py  
- **extract_main_history.py** – читает `labcore_draws.csv`, выделяет основные шары, режим day/evening и сохраняет `main_history.csv`.  
- **extract_bonus_history.py** – аналогично для бонусных шаров.

---

### features/\*.py  
Каждый модуль добавляет свой блок признаков:
1. **recency.py** → `add_recency()`: “age” (возраст номера с последнего выпадения).  
2. **frequencies.py** → `add_sliding_freq()`: `freq_short`, `freq_long`.  
3. **periodicity.py** → `add_periodicity()`: `mod7`, `mod30`, `is_weekend`.  
4. **mode_freq.py** → `add_mode_specific_freq()`: `freq_day`, `freq_evening`.  
5. **structure.py** → `add_structural()`: `sum6`, `var6`, декады, `run_len`, `prime_cnt`, `low_cnt`, `mid_cnt`, `high_cnt`.  
6. **correlations.py** → `add_correlations()`: `pair_hot`, `triplet_hot`.  
7. **markov.py** → `add_markov_transitions()`: любые `trans_*` признаки.  
8. **mode_ohe.py** → `add_mode_ohe()`: one-hot кодировка поля `mode`.

---

### training_main_ensemble.py  
Оркеструет:
1. `extract_main_history()`  
2. Чтение конфига  
3. `build_main_features()` из всех модулей  
4. Разбиение на train/test по `draw`  
5. Обучение `RandomForestClassifier`  
6. Обучение `LogisticRegression` на вероятностях RF  
7. Сохранение моделей в `models/`  
8. Вывод ROC-AUC и Top-K Recall

### training_bonus_ensemble.py  
Аналогично для бонусных шаров (использует `extract_bonus_history.py`).

---

### utils/structure_quota.py  
Функция `compute_structure_quota()`:
- Строит частоты паттернов `low-mid-high` из `main_history.csv`.  
- Читает **history_metrics.csv** (HitRate паттернов).  
- Вычисляет комбинированный вес и нормирует в квоты.

---

### generate_structured.py  
Функция `generate_structured_combinations()`:
- Принимает квоты структуры и модельные вероятности `{num→score}`.  
- Сначала выбирает паттерн `low-mid-high` по квотам.  
- Затем для каждой зоны берёт топ-N номеров по score.  
- Возвращает список отсортированных комбинаций.

---

### ai_controller.py  
Запускает все шаги конвейера в нужном порядке:
1. `python extract_bonus_history.py`  
2. `python extract_main_history.py`  
3. `python training_main_ensemble.py`  
4. `python training_bonus_ensemble.py`

---

## Как воспроизвести

```bash
# из корня проекта:
cp -r . checkpoint_v1
cd checkpoint_v1
pip install -r requirements.txt
python ai_controller.py
```

В консоли появятся метрики, в папке `models/` — сохранённые `.pkl` модели.

---

## Дальнейшие шаги

- Для перехода к v2 создайте `checkpoint_v2` и проделайте ту же процедуру.  
- Все изменения фиксируйте в README и в `requirements.txt`.

---

Теперь, прочитав этот README, любой сможет восстановить структуру каталогов, файлы и порядок выполнения без лишних вопросов.
