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
**Бизнес-задача:** Выявить лучших сотрудников для премирования и передачи опыта.
```sql
select manager, sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_revenue
from orderss
group by manager
order by total_revenue desc;
```
<img width="1008" height="600" alt="2026-11-12_19-48-21" src="https://github.com/user-attachments/assets/ffba9e8c-b2b4-43eb-94eb-984d11655d12" />


**Результат:** Лазарева Кира принесла 52 млн руб., Медведева Злата - 28 млн. Остальные менеджеры принесли менее 10 млн.
**Вывод для бизнеса:** Проанализировать скрипты лидера (Киры) и передать опыт остальным сотрудникам.

## Задача 3: Влияние подарков на средний чек
**Бизнес-задача:** Проверить, заставляют ли подарки клиентов тратить больше.
```sql
select case when gift is null then 'Без подарка' else 'С подарком' end as giff_status,
round(avg(regexp_replace(price, '[^0-9]', '', 'g')::numeric),2) as avg_chec
from orderss
group by giff_status;
```
<img width="1068" height="373" alt="2026-11-12_19-21-21" src="https://github.com/user-attachments/assets/82650983-bb3c-46b8-a9dd-42dd2bd3dcd9" />

**Результат:** Средний чек без подарка - 12 285 руб., с подарком - 12 787 руб. Разница 502р.
**Вывод для бизнеса:** Подарки не стимулируют покупать больше. Акция убыточна, так как затраты на подарки не окупаются ростом чека.

## Задача 4: Анализ затрат на логистику
**Бизнес-задача:** Проверить, не съедает ли доставка маржинальность в удаленных регионах.
```sql
select city, sum(regexp_replace(delivery_cost, '[^0-9]', '', 'g')::numeric) as total_delivery_cost,
sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_revenue,
round(
      (sum(regexp_replace(delivery_cost, '[^0-9]', '', 'g')::numeric)/
	  sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric))*100,2) as percent
from orderss
group by city
order by percent desc;
```
<img width="1492" height="956" alt="2026-11-12_19-22-04" src="https://github.com/user-attachments/assets/9885c5a5-de38-4fa6-b4a6-ceff6167530e" />

**Результат:** Даже в самых дорогих городах логистика занимает менее **1%** от выручки.
**Вывод для бизнеса:** Бизнес может ввести бесплатную доставку как маркетинговый ход, не рискуя уйти в убыток.

## Задача 5: Когортный анализ клиентов (LTV и удержание)
**Бизнес-задача:** Понять, кто приносит больше денег - новые клиенты или постоянные.
```sql
with a as(select client_fio, count(*) as order_count, sum(regexp_replace(price, '[^0-9]', '', 'g')::numeric) as total_spent
from orderss
group by client_fio)
select case when order_count  = 1 then 'Купили 1 раз(новые)'
            when order_count between 2 and 3 then 'Купили 2-3 раза'
			else 'Купили 4+ раз(постоянные)'
			    end as client_category,  round(
        count(*) * 100.0 / (select count(distinct client_fio) from orderss), 
        2) as client_percent,				
count(*) as client_cnt, sum(total_spent) as total_sum,
round((sum(total_spent)/count(*)),2) as avg_check
from a
group by client_category
order by avg_check asc;
```
<img width="1498" height="638" alt="2026-11-12_19-22-51" src="https://github.com/user-attachments/assets/aabdfdeb-ba8b-41d8-888e-b808ec500204" />

**Результат:** Постоянные клиенты (5.66% от базы) принесли 9.5 млн руб. Их средний чек - 36 817 руб., что в 2 раза выше, чем у новых (17 936 руб.).
**Вывод для бизнеса:** Внедрить программу лояльности и скидки на второй заказ, чтобы удерживать новых клиентов. Это принесет больше прибыли, чем привлечение новых, так как новых клиентов больше половины - 54,58%.

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
df = pd.read_csv(r"C:\Users\volod\OneDrive\Рабочий стол\ab_test_data.csv" )
```
<img width="1295" height="455" alt="2026-11-12_19-25-30" src="https://github.com/user-attachments/assets/af264e6a-fd55-4344-9f8f-ff3155d9b6d1" />

  
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

**Результат:** P-value = **0.159942**, что больше 0.05. Разница в среднем чеке статистически незначима.
**Вывод для бизнеса:** Пересмотреть акцию с подарками, либо отказаться от нее, так как она не увеличивает чек, а только снижает маржинальность.
 
## Визуализация (Power BI)
Для наглядности все ключевые метрики собраны в единый интерактивный дашборд.
<img width="1920" height="1014" alt="2026-11-12_18-19-05" src="https://github.com/user-attachments/assets/0ab1de85-d64a-40ad-bd2b-1906867098d6" />


