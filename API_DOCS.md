# API документация — Обмен данными с 1С

**Версия:** 1.0.0  
**Базовый URL:** `https://your-domain.com/api`  
**Формат данных:** JSON  
**Кодировка:** UTF-8

---

## Содержание

1. [Авторизация](#авторизация)
2. [Общие правила](#общие-правила)
3. [Эндпоинты](#эндпоинты)
   - [Импорт остатков](#1-импорт-остатков)
   - [Экспорт остатков](#2-экспорт-остатков)
   - [Health-check остатков](#3-health-check-остатков)
   - [Импорт заказов](#4-импорт-заказов)
   - [Экспорт заказов](#5-экспорт-заказов)
   - [Общий health-check](#6-общий-health-check-обмена)
4. [Коды ответов](#коды-ответов)
5. [Примеры ошибок](#примеры-ошибок)
6. [Ограничения](#ограничения)

---

## Авторизация

API использует авторизацию по ключу в HTTP-заголовке.

### Заголовок

```
X-Api-Key: <ваш-ключ>
```

Ключ передаётся в каждом запросе к защищённым эндпоинтам.

### Пример

```bash
curl -H "X-Api-Key: ваш-секретный-ключ" https://your-domain.com/api/stocks/import
```

### Публичные эндпоинты (без ключа)

Следующие эндпоинты доступны **без авторизации**:

| Эндпоинт | Назначение |
|----------|------------|
| `GET /api/stocks/health` | Проверка работоспособности API остатков |
| `GET /api/exchange/health` | Общая проверка работоспособности обмена |

### Ошибки авторизации

**Ключ не передан:**

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json; charset=utf-8
```

```json
{
  "success": false,
  "message": "Access denied. API key is required (X-Api-Key header)."
}
```

**Неверный ключ:**

```json
{
  "success": false,
  "message": "Access denied. Invalid API key."
}
```

---

## Общие правила

1. Все запросы и ответы — в формате **JSON** (`Content-Type: application/json`)
2. Все ответы содержат поле `"success": true|false` и `"message"`
3. Для POST-запросов данные передаются в теле запроса
4. Максимальный размер тела запроса — **5 МБ**
5. Кодировка — **UTF-8**

### Формат успешного ответа

```json
{
  "success": true,
  "message": "Описание результата",
  ...дополнительные поля
}
```

### Формат ответа с ошибкой

```json
{
  "success": false,
  "message": "Описание ошибки"
}
```

---

## Эндпоинты

### 1. Импорт остатков

Приём и обновление остатков товаров по складам.

```
POST /api/stocks/import
```

**Авторизация:** обязательна (`X-Api-Key`)

#### Тело запроса

Массив объектов складов. Каждый склад содержит код, название и массив товаров с размерами.

```json
[
  {
    "codeSklad": "string — код склада",
    "nameSklad": "string — название склада",
    "products": [
      {
        "codeProduct": "string|number — SKU товара",
        "nameProduct": "string — название товара",
        "descriptionProduct": [
          {
            "namedescription": "string — описание варианта",
            "size": "string — размер",
            "count": "number — количество"
          }
        ]
      }
    ]
  }
]
```

#### Описание полей

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|----------|
| `codeSklad` | string | ✅ | Уникальный код склада в 1С (например, `"00-000001"`) |
| `nameSklad` | string | ❌ | Человекочитаемое название склада |
| `products` | array | ✅ | Массив товаров на данном складе |
| `products[].codeProduct` | string/number | ✅ | SKU товара (соответствует полю `sku` в БД) |
| `products[].nameProduct` | string | ❌ | Название товара (информационное поле) |
| `products[].descriptionProduct` | array | ❌ | Массив размеров с остатками |
| `descriptionProduct[].namedescription` | string | ❌ | Текстовое описание варианта |
| `descriptionProduct[].size` | string | ✅ | Размер (`"40"`, `"41"`, `"XL"` и т.д.) |
| `descriptionProduct[].count` | number | ✅ | Остаток на складе (0 — товар отсутствует) |

#### Пример запроса

```bash
curl -X POST https://your-domain.com/api/stocks/import \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: ваш-секретный-ключ" \
  -d '[
    {
      "codeSklad": "00-000001",
      "nameSklad": "Торговый зал ТЦ \"Глобо\"",
      "products": [
        {
          "codeProduct": "8433",
          "nameProduct": "Ботинки 5065 Скиф олива",
          "descriptionProduct": [
            {
              "namedescription": "Бутекс•Олива•40",
              "size": "40",
              "count": 1
            },
            {
              "namedescription": "Бутекс•Олива•41",
              "size": "41",
              "count": 1
            },
            {
              "namedescription": "Бутекс•Олива•44",
              "size": "44",
              "count": 1
            },
            {
              "namedescription": "Бутекс•Олива•45",
              "size": "45",
              "count": 2
            },
            {
              "namedescription": "Бутекс•Олива•46",
              "size": "46",
              "count": 1
            }
          ]
        }
      ]
    }
  ]'
```

#### Успешный ответ

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

```json
{
  "success": true,
  "message": "Stock import completed",
  "processed": 1,
  "updated": 1,
  "not_found": 0,
  "not_found_skus": [],
  "errors": []
}
```

#### Описание полей ответа

| Поле | Тип | Описание |
|------|-----|----------|
| `processed` | number | Общее количество обработанных товаров |
| `updated` | number | Количество успешно обновлённых товаров |
| `not_found` | number | Количество товаров, не найденных по SKU |
| `not_found_skus` | array | Список SKU, которые не были найдены в базе |
| `errors` | array | Список ошибок валидации (текстовые сообщения) |

#### Ответ с ненайденными товарами

Если часть товаров не найдена по SKU, запрос **не падает** — обрабатываются найденные, а ненайденные возвращаются в ответе:

```json
{
  "success": true,
  "message": "Stock import completed",
  "processed": 5,
  "updated": 3,
  "not_found": 2,
  "not_found_skus": ["9999", "8888"],
  "errors": []
}
```

#### Отправка нескольких складов одним запросом

В одном запросе можно передать остатки с нескольких складов:

```json
[
  {
    "codeSklad": "00-000001",
    "nameSklad": "Торговый зал",
    "products": [
      {
        "codeProduct": "8433",
        "descriptionProduct": [
          {"size": "40", "count": 1},
          {"size": "41", "count": 3}
        ]
      }
    ]
  },
  {
    "codeSklad": "00-000002",
    "nameSklad": "Склад-основной",
    "products": [
      {
        "codeProduct": "8433",
        "descriptionProduct": [
          {"size": "40", "count": 5},
          {"size": "42", "count": 2}
        ]
      }
    ]
  }
]
```

#### Обнуление остатков

Чтобы обнулить остаток конкретного размера на складе, передайте `"count": 0`:

```json
[
  {
    "codeSklad": "00-000001",
    "products": [
      {
        "codeProduct": "8433",
        "descriptionProduct": [
          {"size": "40", "count": 0},
          {"size": "41", "count": 0}
        ]
      }
    ]
  }
]
```

#### Важные особенности логики

- Обновляются остатки **только для указанного склада** (`codeSklad`). Остатки других складов **не затрагиваются**.
- Если в запросе от склада пришли не все размеры, размеры, которых нет в запросе, **обнуляются** для этого склада.
- Если `count = 0` — размер удаляется из данных этого склада.
- Суммарные остатки по размерам пересчитываются автоматически по всем складам.
- Если у товара не осталось ни одного остатка > 0, товар помечается как «не в наличии».

- Если у товара не осталось ни одного остатка > 0, товар помечается как «не в наличии».

---

### 2. Экспорт остатков

Получение текущего состояния всех остатков (наличия) товаров на сайте.

```
GET /api/stocks/export
```

**Авторизация:** обязательна (`X-Api-Key`)

#### Пример запроса

```bash
curl https://your-domain.com/api/stocks/export \
  -H "X-Api-Key: ваш-секретный-ключ"
```

#### Успешный ответ

```json
{
  "success": true,
  "message": "Stock export completed",
  "count": 1,
  "stocks": [
    {
      "id": 8433,
      "title": "Ботинки 5065 Скиф олива",
      "artikul": 8433,
      "link": "botinki-5065-skif-oliva",
      "stock": 1,
      "size_chart": {
        "40": 1,
        "41": 1,
        "44": 1,
        "45": 2,
        "46": 1
      },
      "availability": {
        "40": { "00-000001": 1 },
        "41": { "00-000001": 1 },
        "44": { "00-000001": 1 },
        "45": { "00-000001": 2 },
        "46": { "00-000001": 1 }
      },
      "lastmod": 1711651200
    }
  ]
}
```

#### Описание полей ответа

| Поле | Тип | Описание |
|------|-----|----------|
| `count` | number | Общее количество выгруженных SKU |
| `stocks` | array | Массив объектов с данными о товарах и остатках |
| `stocks[].id` | number | Внутренний ID записи SKU (равен artikul) |
| `stocks[].title` | string | Название товара |
| `stocks[].artikul` | number | SKU / Артикул товара (целое число) |
| `stocks[].link` | string | URL-слаг товара |
| `stocks[].stock` | number | Общий флаг наличия (1 — есть в наличии, 0 — нет) |
| `stocks[].size_chart` | object | Суммарные остатки по размерам (ключ — размер, значение — количество) |
| `stocks[].availability` | object | Детализированные остатки по складам (размер -> склад -> кол-во) |
| `stocks[].lastmod` | number | Unix timestamp последнего изменения данных о наличии |

---

### 3. Health-check остатков

Проверка работоспособности API и подключения к базе данных.

```
GET /api/stocks/health
```

**Авторизация:** не требуется

#### Пример запроса

```bash
curl https://your-domain.com/api/stocks/health
```

#### Ответ

```json
{
  "success": true,
  "message": "Stocks API is operational",
  "timestamp": "2026-03-28T18:46:49+03:00",
  "version": "1.0.0",
  "database": "ok",
  "endpoint": "stocks"
}
```

#### Описание полей

| Поле | Тип | Описание |
|------|-----|----------|
| `timestamp` | string | Текущее серверное время (ISO 8601) |
| `version` | string | Версия API |
| `database` | string | Статус подключения к БД (`"ok"` или описание ошибки) |
| `endpoint` | string | Идентификатор проверяемого модуля |

---

### 3. Импорт заказов

> ⚠️ **Эндпоинт в разработке.** Структура запроса будет определена позже.

```
POST /api/orders/import
```

**Авторизация:** обязательна (`X-Api-Key`)

#### Пример запроса

```bash
curl -X POST https://your-domain.com/api/orders/import \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: ваш-секретный-ключ" \
  -d '[]'
```

#### Текущий ответ

```json
{
  "success": true,
  "message": "Orders import endpoint is not implemented yet.",
  "implemented": false,
  "hint": "This endpoint will accept orders from 1C. Schema will be defined later."
}
```

---

### 4. Экспорт заказов

> ⚠️ **Эндпоинт в разработке.** Структура ответа будет определена позже.

```
GET /api/orders/export
```

**Авторизация:** обязательна (`X-Api-Key`)

#### Пример запроса

```bash
curl https://your-domain.com/api/orders/export \
  -H "X-Api-Key: ваш-секретный-ключ"
```

#### Текущий ответ

```json
{
  "success": true,
  "message": "Orders export endpoint is not implemented yet.",
  "implemented": false,
  "orders": [],
  "hint": "This endpoint will return orders for 1C. Schema will be defined later."
}
```

---

### 5. Общий health-check обмена

Общая проверка работоспособности API со списком всех доступных эндпоинтов.

```
GET /api/exchange/health
```

**Авторизация:** не требуется

#### Пример запроса

```bash
curl https://your-domain.com/api/exchange/health
```

#### Ответ

```json
{
  "success": true,
  "message": "Exchange API is operational",
  "timestamp": "2026-03-28T18:46:49+03:00",
  "version": "1.0.0",
  "database": "ok",
  "endpoints": {
    "stocks_import": "POST /api/stocks/import",
    "stocks_export": "GET /api/stocks/export",
    "stocks_health": "GET /api/stocks/health",
    "orders_import": "POST /api/orders/import (stub)",
    "orders_export": "GET /api/orders/export (stub)",
    "exchange_health": "GET /api/exchange/health"
  }
}
```

---

## Коды ответов

| HTTP-код | Название | Когда возвращается |
|:--------:|----------|-------------------|
| **200** | OK | Запрос обработан успешно |
| **403** | Forbidden | API-ключ отсутствует, неверный или деактивирован |
| **404** | Not Found | Запрошенный URL не соответствует ни одному эндпоинту |
| **405** | Method Not Allowed | Эндпоинт существует, но вызван с неверным HTTP-методом |
| **413** | Payload Too Large | Размер тела запроса превышает 5 МБ |
| **422** | Unprocessable Entity | Ошибка валидации: невалидный JSON, некорректная структура данных |
| **500** | Internal Server Error | Непредвиденная ошибка на сервере |

---

## Примеры ошибок

### Невалидный JSON (422)

```bash
curl -X POST https://your-domain.com/api/stocks/import \
  -H "X-Api-Key: ваш-секретный-ключ" \
  -d '{invalid json}'
```

```json
{
  "success": false,
  "message": "Invalid JSON in request body.",
  "errors": {
    "json_error": "Syntax error, malformed JSON"
  }
}
```

### Пустое тело запроса (422)

```json
{
  "success": false,
  "message": "Request body is empty."
}
```

### Некорректная структура данных (422)

```json
{
  "success": false,
  "message": "JSON root must be an array of warehouse objects."
}
```

### Маршрут не найден (404)

```bash
curl https://your-domain.com/api/unknown
```

```json
{
  "success": false,
  "message": "Route GET /api/unknown not found."
}
```

### Неверный HTTP-метод (405)

```bash
curl -X GET https://your-domain.com/api/stocks/import
```

```json
{
  "success": false,
  "message": "Method GET is not allowed for /api/stocks/import. Use POST."
}
```

### Слишком большой запрос (413)

```json
{
  "success": false,
  "message": "Request payload is too large"
}
```

### Ошибка сервера (500)

```json
{
  "success": false,
  "message": "Internal server error"
}
```

---

## Ограничения

| Параметр | Значение |
|----------|----------|
| Максимальный размер тела запроса | 5 МБ |
| Формат данных | JSON |
| Кодировка | UTF-8 |
| Авторизация | API-ключ в заголовке `X-Api-Key` |
| Rate limit | Не ограничен (планируется в будущих версиях) |

---

## Сводная таблица эндпоинтов

| Метод | Путь | Авторизация | Статус |
|:-----:|------|:-----------:|:------:|
| `POST` | `/api/stocks/import` | ✅ | Работает |
| `GET` | `/api/stocks/export` | ✅ | Работает |
| `GET` | `/api/stocks/health` | ❌ | Работает |
| `POST` | `/api/orders/import` | ✅ | Заглушка |
| `GET` | `/api/orders/export` | ✅ | Заглушка |
| `GET` | `/api/exchange/health` | ❌ | Работает |
