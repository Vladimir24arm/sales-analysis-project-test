# Анализ продаж магазина одежды 

**Цель проекта:** Провести комплексный анализ базы заказов (7 771 строка) для выявления точек роста бизнеса, оптимизации маркетинга и оценки эффективности персонала.

**Инструменты:** 
*   **SQL (PostgreSQL):** Очистка, агрегация и когортный анализ данных.
*   **Python (Pandas, SciPy):** Проведение A/B-теста и статистический анализ. 
*   **Power BI:** Построение интерактивного дашборда.

## Задача 1: Топ-10 городов по количеству заказов
**Бизнес-задача:** Понять географию продаж и выявить города-лидеры по количеству заказов.

```sql
select  city, count(*) as cnt 
from orderss
where price != '0'
group by city
order by cnt desc
limit 10;
```
<img width="632" height="685" alt="2026-11-13_18-34-41" src="https://github.com/user-attachments/assets/8be947b1-4d39-4598-a32d-789621572386" />



**Результат:** Лидеры по количеству заказов - Самара (192), Ставрополь (189) и Нижний Новгород (187).
**Вывод для бизнеса:** Анализ показал, что распределение заказов по городам равномерное. Примечательно то, что не самые крупные города показывают результат по заказам на уровне городов милионников. Например, население Ставрополя 570.000 человек, количество заказов 189, население Нижнего Новгорода 1,2млн человек, количество заказов - 187. Рекомендуется проверить эффективность рекламных кампаний в крупных городах.

## Задача 2: Оценка эффективности менеджеров
**Бизнес-задача:** Выявить средний чек каждого сотрудника для выявления эффективности работы сотрудников.
```sql
select manager, sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_revenue,
round(avg(regexp_replace(price, '[^0-9]', '', 'g')::numeric), 2) as avg_check
from orderss
where price != '0'
group by manager
order by total_revenue desc;
```
<img width="1020" height="583" alt="2026-11-13_22-12-42" src="https://github.com/user-attachments/assets/30b38cc3-98f8-476f-b4ba-4f1f94ff663c" />


**Результат:** Посчитал общую выручку по каждому сотруднику и средний чек. Сотрудник с самой большой выручкой - Лазарева Кира (52млн.р), однако самый высокий средний чек у Громовой Алисы (18 142р). Средний чек менеджеров в диапазоне 12000-16000р, что в пределах нормы с небольшими отклонениями в обеих границах.
**Вывод для бизнеса:** Существенной разницы в среднем чеке нет, бизнесу стоит сфокусироваться на увеличении не среднего чека, а на увеличении потока клиентов и количества позиций в каждом чеке для прироста общей выручки.

## Задача 3: Влияние подарков на средний чек
**Бизнес-задача:** Проверить, заставляют ли подарки клиентов тратить больше.
```sql
select case when gift is null then 'Без подарка' else 'С подарком' end as giff_status,
round(avg(regexp_replace(price, '[^0-9]', '', 'g')::numeric),2) as avg_chec, count(*) as cnt
from orderss
where price != '0'
group by giff_status;
```
 <img width="1073" height="386" alt="2026-11-13_22-25-45" src="https://github.com/user-attachments/assets/d3d36103-5a67-40bf-9115-8c292fa68d8c" />


**Результат:** Средний чек без подарка - 13402.94, с подарком - 21078,34 руб. Разница 7675,4 руб.
**Вывод для бизнеса:**  Клиенты, получающие подарок, тратят на 7675,4 руб. больше (разница в 1,5 раза). Это говорит о том, что акция с подарками эффективна и стимулирует покупать более дорогие товары. Рекомендую сохранить текущую политику подарков.

## Задача 4: Анализ затрат на логистику
**Бизнес-задача:** Проверить, не съедает ли доставка маржинальность в удаленных регионах.
```sql
select city, sum(regexp_replace(delivery_cost, '[^0-9]', '', 'g')::numeric) as total_delivery_cost,
sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_revenue,
round(
      (sum(regexp_replace(delivery_cost, '[^0-9]', '', 'g')::numeric)/
	  sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric))*100,2) as percent
from orderss
where price != '0'
group by city
order by percent desc;
```
 <img width="1153" height="961" alt="2026-11-13_22-31-44" src="https://github.com/user-attachments/assets/713f2f37-08d8-43b4-a5bd-67ec11bc6a84" />

**Результат:** Даже в самых дорогих городах логистика занимает менее **1%** от выручки.
**Вывод для бизнеса:** Бизнес может ввести бесплатную доставку как маркетинговый ход, не рискуя уйти в убыток.

## Задача 5: Когортный анализ клиентов (LTV и удержание)
**Бизнес-задача:** Понять, какое количество новых клиентов возвращается в последующих месяцах после первой покупки в течение года.
```sql
select sales_channel,  count(*) as cnt, 
round(sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) * 100.0 / 
     (select sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) FROM orderss), 2) as percent,
sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_spent
from orderss
where price != '0'
group by sales_channel
order by total_spent desc;

with a as ( select client_fio,
                   to_char(to_date(order_date, 'DD.MM.YYYY'), 'MM-YYYY') as order_month,
                   min(to_char(to_date(order_date, 'DD.MM.YYYY'), 'MM-YYYY')) 
                   over (partition by client_fio) AS cohort_month
from orderss
    where price != '0')
select cohort_month AS "Когорта",
       order_month AS "Месяц покупки",
       count(distinct client_fio) as "Клиенты"
from a
group by cohort_month, order_month
order by
    to_date(cohort_month, 'MM-YYYY') asc, 
    to_date(order_month, 'MM-YYYY') asc;
```
<img width="1134" height="944" alt="2026-11-13_21-33-19" src="https://github.com/user-attachments/assets/7c0795ca-ccd3-4261-a779-e514e23168cd" />
<img width="419" height="950" alt="2026-11-13_21-34-12" src="https://github.com/user-attachments/assets/77d885c3-5cce-43ee-a642-5f9689b71007" />
<img width="411" height="953" alt="2026-11-13_21-34-48" src="https://github.com/user-attachments/assets/020b68ad-7e05-4b71-8999-dd43820ad5d2" />

```sql
WITH cohorts AS (SELECT client_fio, MIN(TO_CHAR(TO_DATE(order_date, 'DD.MM.YYYY'), 'YYYY-MM')) 
OVER (PARTITION BY client_fio) AS cohort_month, TO_CHAR(TO_DATE(order_date, 'DD.MM.YYYY'), 'YYYY-MM') AS order_month
FROM orderss
WHERE price != '0'),
cohort_sizes AS (SELECT cohort_month,
    COUNT(DISTINCT client_fio) AS total_clients
    FROM cohorts
    GROUP BY cohort_month),
    returned_clients AS (SELECT cohort_month,
                         COUNT(DISTINCT client_fio) AS returned_clients
    FROM cohorts
    WHERE order_month != cohort_month  
    GROUP BY cohort_month)
SELECT cs.cohort_month AS "Когорта",
       cs.total_clients AS "Всего клиентов",
       COALESCE(rc.returned_clients, 0) AS "Вернулось",
       ROUND(COALESCE(rc.returned_clients, 0) * 100.0 / cs.total_clients, 2) AS "Процент удержания (%)"
FROM cohort_sizes cs
LEFT JOIN returned_clients rc ON cs.cohort_month = rc.cohort_month
order by cs.cohort_month;
```
<img width="1328" height="955" alt="2026-11-13_21-42-12" src="https://github.com/user-attachments/assets/b2f23c2f-efd4-42bd-af71-e8bc7c5193f7" />

**Результат:** Проанализировал удержание клиентов по когортам. Из 576 клиентовб пришедших в январе, вернулось в течение года 47 (8,16%). Из 343, пришедших в феврале, вернулось 37 (10.79%). В марте из 193 вернулось 35 (18.13%). В каждом месяце пришедший клиент считается уникальным(первым) и в списке  новых клиентов, пришедших за следующий месяц текущего года не входят клиенты из предыдущего месяца.
**Вывод для бизнеса:** Самый большой процент возврата клиентов - 15,07% (из тех, что впервые пришли за покупкой в апреле (272), вернулось повторно 41. Для того, чтобы понимать, являются ли такие проценты нормой (средний процент возврата 9,75% без учета октября, тк данные ограничиваются этим месяцем), необходимо понимать внутренние цели и планы компании по возврату(удержанию) клиентов. Рекомендую сравнить эти цифры с планами бизнеса. Если уровень удержания выше, стоит пересмотреть программу лояльности и рассылки для стимулирования повторных продаж. 

## Задача 6: Анализ маркетинговых каналов
**Бизнес-задача:** Понять, откуда приходят самые прибыльные клиенты.
```sql
select sales_channel,  count(*) as cnt, 
round(sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) * 100.0 / 
     (select sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) FROM orderss), 2) as percent,
sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_spent
from orderss
where price != '0'
group by sales_channel
order by total_spent desc;
```
<img width="1117" height="674" alt="2026-11-13_20-44-52" src="https://github.com/user-attachments/assets/fc0acd23-a3f1-4ddf-9f77-7ec7f6b742d3" />


**Результат:** Лидер - ВК (37.59% выручки), далее Инстаграм (15.50%) и Телеграм (13.17%).
**Вывод для бизнеса:** Перераспределить рекламный бюджет в пользу ВК и Инстаграмма, так как они дают наибольший возврат. Так же либо пересмотреть ведение менее прибыльных каналов, либо отказаться от них. 

## Задача 7: A/B-тест (Python) - Влияние подарков на чек
**Бизнес-задача:** Проверить статистически, заставляют ли подарки клиентов тратить больше.
```python
import pandas as pd
from scipy import stats
```
# Загрузка данных
```python
import pandas as pd
df = pd.read_csv(r"C:\Users\volod\OneDrive\Рабочий стол\ab-test.csv")
df.head()
```
<img width="1055" height="461" alt="2026-11-13_22-49-05" src="https://github.com/user-attachments/assets/5b54fd09-3aeb-4115-9bf3-f8984d0ba70e" />

  
```python 
control = df[df['group_name'] == 'Control']['price']
test = df[df['group_name'] == 'Test']['price']
```
# Расчет T-теста
```python 
t_stat, p_value = stats.ttest_ind(control, test, equal_var=False)
``` 
```python
print(f"Средний чек (без подарка): {control.mean():.2f} руб.")
print(f"Средний чек (с подарком): {test.mean():.2f} руб.")
print(f"P-value: {p_value:.4f}")
```
<img width="1486" height="528" alt="2026-11-12_19-27-38" src="https://github.com/user-attachments/assets/c60f9a01-4b02-4ecb-8bfc-17a832e4f551" />

**Результат:** P-value = **0.000000**. Разница статистически незначима, подарки влияют на чек.
**Вывод для бизнеса:** Подарки статистически значимо увеличивают средний чек на 7675,4р. Акция эффективна и рекомендуется к сохранению.
 
## Визуализация (Power BI)
Для наглядности все ключевые метрики собраны в единый интерактивный дашборд.
<img width="1920" height="1014" alt="2026-11-12_18-19-05" src="https://github.com/user-attachments/assets/0ab1de85-d64a-40ad-bd2b-1906867098d6" />


