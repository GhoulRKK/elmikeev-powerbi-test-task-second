# DAX-меры

Все меры вынесены в отдельную таблицу `Меры`. Факты берутся из выгрузок Wildberries API (`public v_report_realization_full`, `public v_sales_full`, `public v_orders`), справочники — из таблиц `Справочник номенклатуры`, `Себестоимость`, `Тарифы и доставка`.

## Содержание

- [Базовые метрики](#базовые-метрики)
- [Затраты и прибыль](#затраты-и-прибыль)
- [Динамика и сравнение периодов](#динамика-и-сравнение-периодов)
- [Топы и география](#топы-и-география)
- [Календарь](#календарь)

---

## Базовые метрики

### Выручка WB
Чистая выручка по отчёту о реализации: продажи минус возвраты.
```dax
Выручка WB =
CALCULATE ( SUM ( 'public v_report_realization_full'[retail_amount] ), 'public v_report_realization_full'[doc_type_name] = "Продажа" )
    - CALCULATE ( SUM ( 'public v_report_realization_full'[retail_amount] ), 'public v_report_realization_full'[doc_type_name] = "Возврат" )
```

### Выручка
Псевдоним `Выручка WB` для графика динамики.
```dax
Выручка = [Выручка WB]
```

### Заказов, шт.
Количество заказанных единиц товара.
```dax
Заказов, шт. = SUM ( 'public v_orders'[quantity] )
```

### Комиссия WB
Сумма комиссии маркетплейса за продажи.
```dax
Комиссия WB = SUM ( 'public v_report_realization_full'[ppvz_sales_commission] )
```

### Комиссия WB (% от продаж)
Комиссия, делённая на сумму продаж с учётом скидки без сторно (`is_storno = 0`). `DIVIDE` защищает от деления на ноль.
```dax
Комиссия WB(% от продаж) =
DIVIDE (
    [Комиссия WB],
    CALCULATE ( SUM ( 'public v_sales_full'[price_with_disc] ), 'public v_sales_full'[is_storno] = 0 )
)
```

### Логистика
Стоимость доставки по отчёту о реализации.
```dax
Логистика = SUM ( 'public v_report_realization_full'[delivery_rub] )
```

---

## Затраты и прибыль

### Валовая прибыль
Выручка минус все затраты по строкам отчёта: вознаграждение и услуги WB (`ppvz_reward`, `ppvz_vw`, `ppvz_vw_nds`), логистика, налог 2% от цены единицы и себестоимость (`RELATED` из справочника `Себестоимость` по штрихкоду) × количество. Затраты по продажам вычитаются, по возвратам — возвращаются обратно.
```dax
Валовая прибыль =
'Меры'[Выручка WB]
    - CALCULATE (
        SUMX (
            'public v_report_realization_full',
            'public v_report_realization_full'[ppvz_reward]
                + 'public v_report_realization_full'[ppvz_vw]
                + 'public v_report_realization_full'[ppvz_vw_nds]
                + 'public v_report_realization_full'[delivery_rub]
                + DIVIDE ( 'public v_report_realization_full'[retail_amount], 'public v_report_realization_full'[quantity] ) * 0.02
                + RELATED ( 'Себестоимость'[Себестоимость] ) * 'public v_report_realization_full'[quantity]
        ),
        'public v_report_realization_full'[doc_type_name] = "Продажа"
    )
    + CALCULATE (
        SUMX (
            'public v_report_realization_full',
            'public v_report_realization_full'[ppvz_reward]
                + 'public v_report_realization_full'[ppvz_vw]
                + 'public v_report_realization_full'[ppvz_vw_nds]
                + 'public v_report_realization_full'[delivery_rub]
                + DIVIDE ( 'public v_report_realization_full'[retail_amount], 'public v_report_realization_full'[quantity] ) * 0.02
                + RELATED ( 'Себестоимость'[Себестоимость] ) * 'public v_report_realization_full'[quantity]
        ),
        'public v_report_realization_full'[doc_type_name] = "Возврат"
    )
```

### Рентабельность
Доля валовой прибыли в выручке.
```dax
Рентабельность = DIVIDE ( [Валовая прибыль], [Выручка WB] )
```

### ROI
Валовая прибыль, делённая на чистую себестоимость проданного (продажи минус возвраты).
```dax
ROI =
DIVIDE (
    [Валовая прибыль],
    CALCULATE (
        SUMX ( 'public v_report_realization_full', RELATED ( 'Себестоимость'[Себестоимость] ) * 'public v_report_realization_full'[quantity] ),
        'public v_report_realization_full'[doc_type_name] = "Продажа"
    )
    - CALCULATE (
        SUMX ( 'public v_report_realization_full', RELATED ( 'Себестоимость'[Себестоимость] ) * 'public v_report_realization_full'[quantity] ),
        'public v_report_realization_full'[doc_type_name] = "Возврат"
    )
)
```

---

## Динамика и сравнение периодов

### Выручка пред. период
Выручка за предыдущий период той же длины, что выбран на фильтре (выбрано 15 дней — берутся предшествующие 15 дней).
```dax
Выручка пред. период =
CALCULATE (
    [Выручка WB],
    DATESBETWEEN (
        'Календарь'[Date],
        MIN ( 'Календарь'[Date] ) - ( DATEDIFF ( MIN ( 'Календарь'[Date] ), MAX ( 'Календарь'[Date] ), DAY ) + 1 ),
        MIN ( 'Календарь'[Date] ) - 1
    )
)
```

### Изменение выручки
Разница текущей выручки и выручки предыдущего периода. На графиках «Лидеры роста» (значения > 0, по убыванию) и «Лидеры падения» (по возрастанию) выводится по артикулам.
```dax
Изменение выручки = 'Меры'[Выручка WB] - [Выручка пред. период]
```

---

## Топы и география

### Возвраты, шт.
Количество возвращённых единиц — для графика «Лидеры возвратов».
```dax
Возвраты, шт. =
CALCULATE ( SUM ( 'public v_report_realization_full'[quantity] ), 'public v_report_realization_full'[doc_type_name] = "Возврат" )
```

### Продажи по областям
Сумма к перечислению продавцу по реализованным продажам; визуал группирует её по `region_name`.
```dax
Продажи по областям =
CALCULATE ( SUM ( 'public v_sales_full'[for_pay] ), 'public v_sales_full'[is_realization] = TRUE () )
```

---

## Календарь

Таблица дат на период данных. «Год-Месяц» в формате `ГГГГ-ММ` сортируется как текст в хронологическом порядке.
```dax
Календарь =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2021, 12, 1 ), DATE ( 2022, 09, 30 ) ),
    "Год", YEAR ( [Date] ),
    "№Месяца", MONTH ( [Date] ),
    "Месяц", FORMAT ( [Date], "MMMM", "ru-RU" ),
    "Год-Месяц", FORMAT ( [Date], "YYYY-MM" ),
    "№Квартала", QUARTER ( [Date] ),
    "Квартал", "Q" & QUARTER ( [Date] ),
    "День", DAY ( [Date] ),
    "День недели", FORMAT ( [Date], "DDDD", "ru-RU" ),
    "№Дня недели", WEEKDAY ( [Date], 2 )
)
```
