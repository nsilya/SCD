Отлично. Делаем **боевой SCD Type 2**, без иллюзий. Ни единого лишнего слова — только рабочий код, который выдержит нагрузку и вопросы на собеседовании.

---

### Шаг 1: Удаляем старую таблицу
```sql
DROP TABLE IF EXISTS public.product_history;
```

---

###  Шаг 2: Создаём правильную SCD2-таблицу
```sql
CREATE TABLE public.product_history (
    product_sk     SERIAL PRIMARY KEY,        -- surrogate key (уникальный для каждой версии)
    product_id     INT NOT NULL,              -- business key (может повторяться)
    product_name   VARCHAR(100) NOT NULL,
    effective_date DATE NOT NULL,
    end_date       DATE NOT NULL DEFAULT '9999-12-31',
    current_flag   BOOLEAN NOT NULL DEFAULT TRUE
);

-- Обеспечиваем, что у одного продукта только одна активная запись
CREATE UNIQUE INDEX CONCURRENTLY IF NOT EXISTS idx_product_history_current
ON public.product_history (product_id) WHERE current_flag = TRUE;
```

> ✅ Почему так:
> - `end_date = '9999-12-31'` — избегаем `NULL`, упрощаем диапазонные запросы.
> - `current_flag` — для быстрого доступа к актуальным данным.
> - Уникальный частичный индекс — **защита от логических ошибок**.

---

###  Шаг 3: Скрипт имитации изменений (3 версии)

```sql
-- Версия 1: начальное состояние (действует с 2020-01-01)
INSERT INTO public.product_history (product_id, product_name, effective_date)
VALUES (1, 'Product A', '2020-01-01');

-- Версия 2: изменение 1 (действует с 2022-07-01)
-- Сначала закрываем текущую запись
UPDATE public.product_history
SET end_date = '2022-06-30',
    current_flag = FALSE
WHERE product_id = 1 AND current_flag = TRUE;

-- Затем вставляем новую
INSERT INTO public.product_history (product_id, product_name, effective_date)
VALUES (1, 'Product A (Updated)', '2022-07-01');

-- Версия 3: изменение 2 (действует с 2024-03-15)
UPDATE public.product_history
SET end_date = '2024-03-14',
    current_flag = FALSE
WHERE product_id = 1 AND current_flag = TRUE;

INSERT INTO public.product_history (product_id, product_name, effective_date)
VALUES (1, 'Product A (Final)', '2024-03-15');
```

> Обрати внимание:  
> - **Каждое изменение = UPDATE + INSERT**.  
> - Дата окончания = дата начала новой версии **минус 1 день**.  
> - Никаких `NULL` в `end_date`.

---

### 🔥 Шаг 4: Проверка — «какой был продукт на дату X?»

```sql
-- Проверка 1: на 2021-01-01 → должна быть "Product A"
SELECT product_name
FROM public.product_history
WHERE product_id = 1
  AND '2021-01-01' BETWEEN effective_date AND end_date;

-- Проверка 2: на 2023-01-01 → должна быть "Product A (Updated)"
SELECT product_name
FROM public.product_history
WHERE product_id = 1
  AND '2023-01-01' BETWEEN effective_date AND end_date;

-- Проверка 3: на сегодня (2025-12-05) → должна быть "Product A (Final)"
SELECT product_name
FROM public.product_history
WHERE product_id = 1
  AND CURRENT_DATE BETWEEN effective_date AND end_date;

-- Проверка 4: текущая версия (по флагу)
SELECT product_name, effective_date, end_date
FROM public.product_history
WHERE product_id = 1 AND current_flag = TRUE;
```

✅ Ожидаемый результат:
- 2021 → `'Product A'`
- 2023 → `'Product A (Updated)'`
- Сегодня → `'Product A (Final)'`
- `current_flag = TRUE` → одна строка с `'Product A (Final)'`

---
