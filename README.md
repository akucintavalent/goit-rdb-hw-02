# goit-rdb-hw-02

Домашнє завдання з проєктування реляційної бази даних для ML-сценарію **Customer Churn Prediction**.

## Мета роботи

Мета проєкту — спроєктувати структуру даних для ML-системи прогнозування відтоку клієнтів, реалізувати її у PostgreSQL та продемонструвати коректне формування навчальних даних із дотриманням **point-in-time correctness**.

Як доменний референс використано датасет **IBM Telco Customer Churn**. Він є snapshot-набором, тому не містить точної часової історії зміни ознак або точного `churn_timestamp`. Temporal-історія для демонстрації point-in-time correctness створена окремо як контрольований набір даних.

## Структура схеми

Схема містить три основні бізнесові сутності:

- `hw2_customers` — базова інформація про клієнта;
- `hw2_subscriptions` — інформація про контракт та спосіб оплати;
- `hw2_customer_services` — набір послуг, якими користується клієнт.

Окремо реалізовано компоненти feature store:

- `hw2_customer_features_offline` — історичні значення ознак із `valid_from` / `valid_to`;
- `hw2_customer_features_online` — поточні значення ознак для швидкого lookup;
- `hw2_training_events` — моменти прогнозування та цільова змінна `churn_in_horizon`.

## Нормалізація

Бізнесові таблиці `hw2_customers`, `hw2_subscriptions`, `hw2_customer_services` та `hw2_training_events` спроєктовані відповідно до принципів 3НФ.

Offline- та online-feature tables навмисно денормалізовані. Вони містять готовий набір ML-ознак, щоб:

- зменшити кількість `JOIN`;
- забезпечити швидкий lookup за `customer_id`;
- зменшити latency під час online prediction;
- використовувати однаково визначені features під час training та serving.

## Point-in-Time correctness

Історичні ознаки зберігаються в `hw2_customer_features_offline` на напіввідкритих часових інтервалах:

```text
[valid_from, valid_to)
```

де:

- `valid_from` входить до інтервалу;
- `valid_to` не входить до інтервалу;
- `valid_to = NULL` означає поточну версію.

Для формування training data використовується AS-OF JOIN:

```sql
f.valid_from <= e.prediction_ts
AND (
    f.valid_to > e.prediction_ts
    OR f.valid_to IS NULL
)
```

Таким чином до кожної training event приєднуються лише ті ознаки, які були відомі на момент `prediction_ts`, що запобігає **data leakage**.

## Реалізовано

У notebook реалізовано:

- ER-діаграму;
- DDL для всіх таблиць;
- `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL` та `CHECK`;
- temporal constraint для `valid_from` / `valid_to`;
- індекси для основних access patterns;
- тестові дані для бізнесових сутностей;
- щонайменше 10 історичних feature-записів для трьох клієнтів;
- щонайменше три temporal-версії для одного клієнта;
- три AS-OF запити;
- AS-OF JOIN для формування training data;
- перевірку перекривних temporal-інтервалів;
- пояснення нормалізації та денормалізації;
- Reflection та підсумкові висновки.

## Файли репозиторію

```text
goit-rdb-hw-02/
├── goit_rdb_hw_02.ipynb
├── ER-diagram.png
└── README.md
```


## Запуск

Робота розрахована на виконання у **Google Colab**.

1. Відкрити `goit_rdb_hw_02.ipynb` у Google Colab.
2. Запустити notebook через **Runtime → Restart session and run all**.
3. Переконатися, що всі SQL-команди виконуються без помилок.
4. Перевірити, що запит на пошук перекривних temporal-інтервалів повертає `0` рядків.

DDL написаний із використанням синтаксису та типів даних, сумісних із **PostgreSQL 16**.

## Доменний датасет

IBM Telco Customer Churn використовується як джерело понять та атрибутів предметної області.

Notebook завантажує CSV із публічного репозиторію IBM та аналізує його структуру. Оскільки датасет є snapshot-набором, temporal-записи для offline feature store створені окремо для демонстрації історії ознак.
