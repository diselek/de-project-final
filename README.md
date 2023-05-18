# Итоговый проект

# Я.Практикум, DE, Итоговый проект

# I. Схема пайплайна

## 0. Сервис Практикума будет эмулировать реальную жизнь, отправляя поток мне в Кафку в Я.Облаке

## 1. Спаркоджоб будет приземлять стрим в две таблицы постгреса as is
(`payload` просто строкой), с некоторыми зачатками дедупликации с ватермарком 5 минут.

*Он просто висит резидентом в контейнере. На прод - наверное в под-е кубера или т.п., и то на прод, я так понимаю, приземление потока работало бы без посредничества постгреса, сразу в Вертику.*

`src/py/stage_stream.py`

Доступы к Fafka, Постгресу, имя топика в Кафке, структура таблицы в Постгресе - всё зашито в код.

Ниже подробно описано, как запускал стриминговый сервис Практикума, как запускал этого воркера в контейнере.

## 2. Один DAG под Airflow: запускается на даты октября 2022-го года backfill-ом

`src/dags/project_final.py`

Параметры бэкфилла:

```python
from airflow.decorators import dag, task
# ...
default_args = {
    'owner': 'student',
    'email': ['student@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    'depends_on_past': True,  # допустим, это надо для корректности витрин
    'wait_for_downstream': True
}
# ...
@dag(
    default_args=default_args,
    start_date=pendulum.parse('2022-10-02'),
    end_date=pendulum.parse('2022-11-01'),
    catchup=True,
    schedule="@daily",
)

```

Граф:

```python
    chain(
        init(),
        [dump_currencies(), dump_transactions()],
        [load_currencies(), load_transactions()],
        make_dwh(),  # pass
        make_cdm()
    )
```

Доступы к Vertica DAG будет искать в Connections Airflow по имени `vertica_connection`,
а доступы к Postgresql будет искать в Connections Airflow по имени `pg_connection`.

Во все task-и передаётся `execution_date`, работа идёт по дате `-1 день` от этой даты (сегодня отрабатываем стрим за вчера, так как сегодня он уже (допущение) заполнен полностью).

Для модуляризации классы-репозитории для работы с Постгресом и Вертикой вынесены в модуль, и вообще структуру сделал согласно [мануалу Airflow](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/modules_management.html):

```bash
/lessons/dags
| .airflowignore # project_final/db_reps/.*
| project_final
    | __init__.py
    | db_reps
    |    |  __init__.py
    |    | PgRepository.py
    |    | VerticaRepository.py
    | dags
        | project_final.py
```

**NB:** Импорты модуля `project_final.db_reps` Airflow заспознал только после перезапуска.

### 2.1. init()
Создаёт таблицы в Вертике.

### 2.2. dump_...(), load_...(): Перекидываем поток, "приземлённый" в Постгрес as is, в стейдж Вертики с раскладкой payload по полям.

Отрабатывает наш DAG по датам, и в Вертике партиционирование по датам же мы делаем.

Соответственно вся работа с данными реализована цепочкам по схеме удаления партиции, заливки партиции.

Стейджинг таблиц Постгреса в таблицы Вертики делаем через файл: удалили партицию в Вертике, выгрузили данные из Постгреса в файл, залили файл в Вертику.

### 2.3. make_dwh(),  `# pass`: Ядро DWH - не делаем

Начал было делать, придумал снежинку, но увидел, что по ТЗ проекта - не требуется.

Ниже есть намётки.

### 2.4. make_cdm()

Записи в витрину добавляем в том же DAG-e, по той же схеме: удаляем партицию по дате, добавляем записи за дату.

На всякий случай в сиквеле предусмотрено так же WHERE PK NOT IN target, как делали в модуле про Вертику.


## 3. Скриншоты дашбоардов в BI

`src/img`

Исходные сырые данные в Постгресе у меня в итоге залились вот так:

```sql
SELECT
    (cast("payload" AS json)->>'transaction_dt')::date AS "anchor_date",
    COUNT(*) AS "cnt"
FROM "stg"."transactions" "t"
GROUP BY (cast("payload" AS json)->>'transaction_dt')::date
ORDER BY (cast("payload" AS json)->>'transaction_dt')::date ASC
;

2022-10-01 | 87098
2022-10-05 | 88168
2022-10-08 | 24129
2022-10-10 | 4811
2022-10-18 | 2070
2022-10-20 | 38665
```

Соответственно им DAG и построил тасками витрину в Вертике, и соответственно Metabase её отрисовал.
