# API Документация Way2Pay

## Введение

Добро пожаловать в документацию по API **Way2Pay**. Наш API  предназначен для безопасного взаимодействия между внешними системами и сервисами платформы. Он предоставляет доступ к операциям по управлению транзакциями, проверке баланса, созданию платёжных форм и выполнению выплат.

## Аутентификация и Подпись

Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью приватного ключа (PrivateKey), который мы предоставляем нашим клиента в Личном Кабинете. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи

Подпись генерируется с использованием алгоритма HMAC-SHA512. Ниже приведен пример функции генерирования подписи (Signature):

#### JavaScript/Node.js
```javascript
const crypto = require('crypto');

// Функция сортировки объекта по ключам
function sortObjectKeys(obj) {
  if (obj === null || typeof obj !== 'object' || Array.isArray(obj)) {
    return obj;
  }
  
  const sortedObj = {};
  const keys = Object.keys(obj).sort();
  
  for (const key of keys) {
    sortedObj[key] = sortObjectKeys(obj[key]);
  }
  
  return sortedObj;
}

function generateSignature(path, body, nonce, privateKey) {
  // Для POST запросов сортируем ключи в body
  let bodyString = '';
  if (body && typeof body === 'object') {
    const sortedBodyObj = sortObjectKeys(body);
    bodyString = JSON.stringify(sortedBodyObj);
  }
  
  // Формируем строку для подписи: path + body + nonce
  const stringToSign = path + bodyString + nonce;
  
  // Генерируем подпись HMAC-SHA512
  const signature = crypto.createHmac('sha512', privateKey)
    .update(stringToSign)
    .digest('hex');
  
  return {
    stringToSign,
    signature,
    body: bodyString
  };
}

// Пример использования для GET запроса
const path = '/api/v1/balance';
const body = null; // Пустое тело для GET запроса
const nonce = generateNonce(); // Уникальное значение
const privateKey = 'your_private_key_here';

const result = generateSignature(path, body, nonce, privateKey);
console.log('String to sign:', result.stringToSign);
console.log('Signature:', result.signature);

// Пример для POST запроса
const postPath = '/api/v1/pay-in';
const postBody = {
  amount: "1000",
  bankId: 1,
  callbackURL: "https://test.com/callback",
  currencyId: 1,
  externalID: "test123",
  method: "CARD"
};
const postResult = generateSignature(postPath, postBody, nonce, privateKey);
console.log('POST String to sign:', postResult.stringToSign);
console.log('POST Signature:', postResult.signature);
```

#### Важные особенности NONCE

**Проблема с NONCE:**
- Система запоминает использованные NONCE для каждого мерчанта в поле `lastNonce` (используется для защиты от replay-атак)
- Повторное использование NONCE приводит к ошибке `invalid NONCE (код 2007)`
- NONCE должен быть больше `lastNonce` в базе данных
- Каждый NONCE может быть использован только один раз

**Решение:**
```javascript
let counter = 0;

function generateNonce() {
  const timePart = Date.now(); // 13 цифр (мс до 2286)
  const counterPart = (counter++ % 1000).toString().padStart(3, "0"); // 3 цифры
  const randomPart = Math.floor(Math.random() * 100).toString().padStart(2, "0"); // 2 цифры
  
  return parseInt(`${timePart}${counterPart}${randomPart}`);
}

const nonce = generateNonce();
```

##### Принцип работы
Функция генерирует уникальный идентификатор на основе текущего времени в миллисекундах, счётчика и случайного числа. Это обеспечивает высокую степень уникальности генерируемых значений.

##### Структура nonce
```
TTTTTTTTTTTTTCCC RR
├─ Время (13 цифр) - текущее время в миллисекундах (от 2286 года)
├─ Счетчик (3 цифры) - счётчик (сбрасывается каждый день)
└─ Случайность (2 цифры) - случайное число (от 0 до 99)
```

##### Ключевые параметры
- **Время (timePart):** текущее время в миллисекундах (13 цифр).
- **Счётчик (counterPart):** трёхзначный счётчик, который увеличивается с каждым вызовом функции и сбрасывается после достижения 999.
- **Случайность (randomPart):** двухзначное случайное число (0-99), добавляющее дополнительную энтропию.

##### Возвращаемое значение
Функция возвращает целое число в формате BIGINT UNSIGNED (до 18 цифр), совместимое с большинством СУБД. Пример: ***172325680000000112***.

##### Преимущества подхода
- **Высокая уникальность:**
  - 1,000,000 уникальных значений/мс (1,000 счётчик × 100 случайных)
  - Поддержка до 1,000 RPS без коллизий

- **Совместимость:**
  - Работает с MySQL, PostgreSQL, Redis
  - Автоматически конвертируется в BIGINT

- **Простота реализации:**
  - Не требует синхронизации между серверами
  - Минимальные накладные расходы

##### Минусы использования простого timestamp:
- Не гарантирует уникальность при параллельных запросах
- Может нарушать хронологический порядок
- Уязвим к атакам повторного воспроизведения
- Не масштабируется под высокие нагрузки
- При множественных запросах в одну секунду возникают коллизии
- Отсутствие дополнительной энтропии
- Высокий риск повторного использования NONCE

#### Как работает валидация
1. Система сравнивает новый NONCE с lastNonce в базе данных
2. Новый NONCE должен быть строго больше предыдущего
3. При успешной валидации lastNonce обновляется в базе

### Формирование Сообщения

Сообщение (stringToSign), используемое для генерации подписи, формируется одинаково для всех типов запросов:

**Формула**: `path + body + nonce`

- **path** - путь к эндпоинту (например: `/api/v1/balance`)
- **body** - JSON строка тела запроса (пустая строка для GET запросов)
- **nonce** - уникальное числовое значение

**⚠️ ВАЖНО**: Значение `nonce` ВСЕГДА добавляется в конец сообщения для подписи.

#### <span style="color:red">
*JSON ключи в теле (body) и query параметры запроса должны идти в алфавитном порядке!* </span>

#### Пример формирования сообщения для GET запроса:
- **URL**: `/api/v1/balance`
- **nonce**: `1721585422` - уникальное числовое значение
- **Сообщение для подписи**: `/api/v1/balance1721585422`

#### Пример формирования сообщения для POST запроса:
- **path**: `/api/v1/pay-in`
- **nonce**: `1721585422`
- **Тело запроса**: `{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}`
- **Сообщение для подписи**: `/api/v1/pay-in{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}1721585422`

### Обязательные заголовки

Каждый запрос к API должен включать следующие заголовки:

- **Content-Type**: `application/json`
- **Public-Key**: Ваш публичный ключ, предоставленный *Way2Pay*
- **nonce**: Уникальное числовое значение для предотвращения повторных запросов (должно быть больше предыдущего)
- **Signature**: Подпись HMAC-SHA512, сгенерированная с использованием вашего приватного ключа

#### Пример заголовков

```http
Content-Type: application/json
nonce: 1717025133
Public-Key: your_public_key_here
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

## API Endpoints

### Основные эндпоинты

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/api/v1/balance` | Получение баланса |
| GET | `/api/v1/banks` | Получение списка банков |
| GET | `/api/v1/currencies` | Получение списка валют |
| GET | `/api/v1/commissions` | Получение комиссий |
| POST | `/api/v1/pay-in` | Создание заявки на прием платежа |
| GET | `/api/v1/pay-in/list` | Получение списка заявок PayIn |
| GET | `/api/v1/pay-in/{id}` | Получение заявки PayIn по ID |
| PUT | `/api/v1/pay-in/{id}/status/{status}` | Обновление статуса заявки PayIn |
| POST | `/api/v1/pay-out` | Создание выплаты |
| GET | `/api/v1/pay-out/list` | Получение списка заявок PayOut |
| GET | `/api/v1/pay-out/{id}` | Получение заявки PayOut по ID |
| PUT | `/api/v1/pay-out/{id}/status/{status}` | Обновление статуса заявки PayOut |

## Выполнение Запросов

### GET Запрос

Пример GET запроса для получения баланса:

```http
GET /api/v1/balance HTTP/1.1
Host: api.way2pay.top
Content-Type: application/json
nonce: 1717025133
Public-Key: your_public_key_here
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

### POST Запрос

Пример POST запроса для создания PayIn:

```http
POST /api/v1/pay-in HTTP/1.1
Host: api.way2pay.top
Content-Type: application/json
nonce: 1717025134
Public-Key: your_public_key_here
Signature: 3336894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677

{
  "bankId": 1,
  "externalID": "test_merchant_id_2",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payment",
  "description": "Test payment",
  "amount": "1000",
  "method": "CARD"
}
```

## Ответы API

Все ответы от API Way2Pay возвращаются в формате JSON. Ответ включает в себя:

- **success** - Статус запроса (true/false)
- **data** - Данные, возвращенные API (только в случае успешного ответа)
- **error** - Данные об ошибке (только в случае ошибочного ответа)

### Пример успешного ответа

```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "RUB",
        "available": "10260.76",
        "frozen": "0"
      },
      "payout": {
        "currency": "RUB",
        "available": "9750.50",
        "frozen": "500.00"
      }
    }
  }
}
```

### Пример ответа с ошибкой

```json
{
  "success": false,
  "error": {
    "message": "Invalid Signature",
    "code": 2005
  }
}
```

## Детальное описание API эндпоинтов

### 1. Получение баланса

**GET** `/api/v1/balance`

Получение информации о балансе клиента.

#### Параметры запроса
Нет параметров

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "RUB",
        "available": "10260.76",
        "frozen": "0"
      },
      "payout": {
        "currency": "RUB",
        "available": "9750.50",
        "frozen": "500.00"
      }
    }
  }
}
```

### 2. Получение списка банков

**GET** `/api/v1/banks`

Получение списка доступных банков для проведения операций.

#### Заголовки
- `X-Environment`: SANDBOX | TEST | PRODUCTION (опционально, по умолчанию PRODUCTION)

#### Пример ответа
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "МежБанк",
      "key": "ANY_BANK",
      "currency": "RUB",
      "environment": "PRODUCTION"
    },
    {
      "id": 2,
      "name": "СБП",
      "key": "SBP",
      "currency": "RUB",
      "environment": "PRODUCTION"
    }
  ]
}
```

### 3. Получение списка валют

**GET** `/api/v1/currencies`

Получение списка поддерживаемых валют.

#### Пример ответа
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Рубль",
      "key": "RUB",
      "isActive": true,
    },
    {
      "id": 2,
      "name": "Доллар США",
      "key": "USD",
      "isActive": true
    }
  ]
}
```

### 4. Получение комиссий

**GET** `/api/v1/commissions`

Получение информации о комиссиях (доступно только для ADMIN и SUPER_ADMIN).

#### Заголовки
- `X-Environment`: SANDBOX | TEST | PRODUCTION (опционально)

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "payIn": [
      {
        "id": 1,
        "bank": "МежБанк",
        "percent": 10.6,
        "minAmount": "1000",
        "maxAmount": "100000",
        "isActive": true
      },
      {
        "id": 2,
        "bank": "SBP",
        "percent": 10.6,
        "minAmount": "1000",
        "maxAmount": "200000",
        "isActive": true
      }
    ],
    "payOut": []
  }
}
```

### 5. Создание заявки на прием платежа (PayIn)

**POST** `/api/v1/pay-in`

Создание новой заявки на прием платежа.

#### Параметры запроса

| Параметр | Тип | Обязательный | Описание | Пример |
|----------|-----|--------------|----------|----------|
| bankId | number | Да | ID банка | 1 |
| externalID | string | Да | Уникальный ID в системе мерчанта (1-64 символа) | "test_merchant_id_2" |
| currencyId | number | Да | ID валюты | 1 |
| callbackURL | string | Нет | URL для получения callback | "https://example.com/callback" |
| description | string | Нет | Описание платежа | "Test payment" |
| amount | string | Да | Сумма платежа | "1000" |
| method | string | Да | Метод платежа: CARD, SBP, ACCOUNT, NSPK, CROSSBORDER_CARD, CROSSBORDER_SBP | "CARD" |

#### Пример запроса
```json
{
  "bankId": 1,
  "externalID": "test_merchant_id_2",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payment",
  "description": "",
  "amount": "6543",
  "method": "CARD"
}
```

#### Описание полей:
- `bankId` (number) - ID банка
- `externalID` (string) - Ваш уникальный ID заявки (1-64 символа, только латинские буквы, цифры, дефис и подчеркивание)
- `currencyId` (number) - ID валюты
- `callbackURL` (string, optional) - URL для получения callback уведомлений
- `description` (string, optional) - Описание платежа
- `amount` (string) - Сумма транзакции (число с не более чем 2 знаками после запятой)
- `method` (string) - Метод перевода: `CARD`, `SBP`, `ACCOUNT`, `NSPK`, `CROSSBORDER_CARD`, `CROSSBORDER_SBP`

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_merchant_id_2",
    "trackerId": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PROCESSING",
    "amount": "1000",
    "commission": "1044.20",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2200154965960000",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая оплата",
    "createdAt": "2025-01-01T12:00:00Z",
    "updatedAt": "2025-01-01T12:00:00Z"
  }
}
```

### 6. Создание выплаты (PayOut)

**POST** `/api/v1/pay-out`

Создание новой заявки на выплату.

#### Параметры запроса

| Параметр | Тип | Обязательный | Описание | Пример |
|----------|-----|--------------|----------|----------|
| externalID | string | Да | Уникальный ID в системе мерчанта | "test_payout_123" |
| bankId | number | Да | ID банка | 1 |
| method | string | Да | Метод выплаты: CARD, SBP, ACCOUNT | "CARD" |
| currencyId | number | Да | ID валюты | 1 |
| callbackURL | string | Да | URL для получения callback | "https://example.com/callback" |
| amount | string | Да | Сумма выплаты | "5000" |
| receiver | string | Да | Реквизиты получателя | "4000000000000000" |
| holder | string | Да | Имя получателя | "Иванов Иван Иванович" |

#### Пример запроса
```json
{
  "externalID": "test_payout_123",
  "bankId": 1,
  "method": "CARD",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payout",
  "amount": "5000",
  "receiver": "4000000000000000",
  "holder": "Иванов Иван Иванович"
}
```

#### Описание полей:
- `externalID` (string) - Ваш уникальный ID заявки (1-64 символа, только латинские буквы, цифры, дефис и подчеркивание)
- `bankId` (number) - ID банка
- `method` (string) - Метод перевода: `CARD`, `SBP`, `ACCOUNT`
- `currencyId` (number) - ID валюты
- `callbackURL` (string) - URL для получения callback уведомлений
- `amount` (string) - Сумма выплаты (число с не более чем 2 знаками после запятой)
- `receiver` (string) - Реквизит получателя (номер карты/телефон/счет)
- `holder` (string) - Имя получателя (3-100 символов)

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_payout_123",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PROCESSING",
    "amount": "5000",
    "commission": "355",
    "currency": "RUB",
    "bank": "OZON",
    "method": "CARD",
    "receiver": "4000000000000000",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая выплата",
    "createdAt": "2025-01-01T12:00:00Z"
  }
}
```

### 7. Получение информации о конкретной заявке
#### Получение заявки PayIn по ID

**GET** `/api/v1/pay-in/{id}`

#### Параметры URL
- `id` - ID заявки в системе Way2Pay

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "test_merchant_id_1",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "COMPLETED",
    "amount": "1000",
    "commission": "1044.20",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2202206212345678",
    "holder": "IVAN IVANOV",
    "description": "Тестовая оплата",
    "createdAt": "2024-01-01T12:00:00Z",
    "updatedAt": "2024-01-01T12:05:00Z"
  }
}
```

### 8. Получение заявки PayOut по ID

**GET** `/api/v1/pay-out/{id}`

#### Параметры URL
- `id` - ID заявки

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "test_power_9",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PENDING",
    "amount": "12165",
    "commission": "973.2",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2100153962960000",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая выплата",
    "createdAt": "2025-05-26T20:55:13.968821Z",
    "updatedAt": "2025-05-26T23:55:15.127007+03:00",
  }
}
```

### 9. Получение списка заявок

#### Получение списка PayIn заявок

**GET** `/api/v1/pay-in/list`

#### Получение списка PayOut заявок

**GET** `/api/v1/pay-out/list`

### 10. Обновление статуса заявки

#### Обновление статуса PayIn

**PUT** `/api/v1/pay-in/{id}/status/{status}`

#### Параметры URL для обновления статуса
- `id` - ID заявки в системе Way2Pay
- `status` - Новый статус заявки

#### Строка для подписи
Формат: {URL_PATH}{BODY}{EXPIRES}

- Пример: /api/v1/pay-in/{ID}/status/{STATUS}{}{EXPIRES}

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_merchant_id_2",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "COMPLETED",
    "amount": "1000",
    "currency": "RUB",
    "method": "CARD",
    "createdAt": "2024-01-01T12:00:00Z",
    "updatedAt": "2024-01-01T12:05:00Z"
  }
}
```

## Статусы транзакций

### PayIn статусы
- `CREATED` - Заявка создана
- `PENDING` - В ожидании реквизитов
- `PROCESSING` - Заявка обрабатывается
- `COMPLETED` - Заявка выполнена
- `TIMEOUT` - Истекло время ожидания оплаты
- `CANCELLED` - Заявка отменена
- `ERROR` - Ошибка при обработке
- `INCORRECT_AMOUNT` - Некорректная сумма
- `REFUNDED` - Возвращен (выполнен откат сделки из состояния COMPLETED)

### PayOut статусы
- `CREATED` - Заявка создана
- `PENDING` - Заявка в обработке
- `PROCESSING` - Заявка обрабатывается
- `COMPLETED` - Заявка выполнена
- `TIMEOUT` - Истекло время ожидания оплаты
- `CANCELLED` - Заявка отменена
- `ERROR` - Ошибка при обработке
- `INCORRECT_AMOUNT` - Некорректная сумма
- `REFUNDED` - Возвращен (выполнен откат сделки из состояния COMPLETED)

## Обработка Ошибок

В случае ошибки ответ будет включать:

- **message**: Описание ошибки
- **code**: Код ошибки

### Коды ошибок

#### Ошибки аутентификации (2000-2999)
| Код ошибки | Сообщение | HTTP статус |
| 2005 | invalid Signature | 401 |
| 2007 | invalid NONCE | 401 |

#### Общие ошибки (10000-19999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 10000 | unauthorized | 401 |


#### Ошибки валидации (20000-29999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 20000 | wrong input | 400 |
| 20001 | can't bind body to request model | 422 |
| 20002 | can't bind query parameters | 422 |
| 20003 | failed to parse key | 422 |
| 20004 | signature header value missing or malformed | 400 |
| 20005 | public-Key header value missing or malformed | 400 |
| 20006 | nonce header value missing or outdated | 400 |
| 20012 | invalid query params | 400 |
| 20015 | conflict | 409 |
| 20016 | empty external ID | 400 |

#### Ошибки доступа (30000-39999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 30000 | forbidden | 403 |
| 30001 | no access to requested session | 403 |
| 30002 | requested sessions has expired | 403 |
| 30003 | user doesn't exists | 403 |
| 30004 | zero balance | 403 |
| 30005 | not enough balance | 402 |
| 30006 | amount less than min | 400 |
| 30007 | amount greater than max | 400 |

#### Внутренние ошибки (40000-49999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 40000 | internal error | 500 |

#### Ошибки ресурсов (60000-69999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 60003 | empty Public-Key | 401 |
| 60004 | empty nonce | 401 |
| 60005 | empty Signature | 401 |
| 60007 | request timeout | 408 |
| 60008 | invalid Public-Key | 400 |
| 60009 | empty external ID | 400 |
| 60010 | external ID already exists | 409 |
| 60011 | payment doesn't exists | 404 |
| 60012 | payment is finalized | 409 |
| 60013 | commission doesnt exists | 400 |
| 60014 | bank doesnt exists | 400 |
| 60015 | method doesnt exists | 400 |


## Тестовые окружения

Система поддерживает работу с различными окружениями через заголовок `X-Environment`:

- **SANDBOX** - песочница для тестирования
- **TEST** - тестовое окружение
- **PRODUCTION** - продакшн окружение (по умолчанию)

### Использование

Добавьте заголовок в ваши запросы:
```http
X-Environment: SANDBOX
```

### Поддерживаемые модули

#### Методы для PayIn:
- **CARD** - Платежи банковскими картами
- **SBP** - Система быстрых платежей
- **ACCOUNT** - Переводы на банковские счета
- **NSPK** - Платежи через QR код (НСПК)
- **CROSSBORDER_CARD** - Международные переводы банковскими картами
- **CROSSBORDER_SBP** - Международные переводы СБП

#### Методы для PayOut:
- **CARD** - Выплаты на банковские карты
- **SBP** - Выплаты через Систему быстрых платежей
- **ACCOUNT** - Выплаты на банковские счета

## Callback уведомления

Система отправляет POST запросы на указанный `callbackURL` при изменении статуса транзакции.

### Структура callback для PayIn:

```json
{
  "id": "e42e0768-d913-4b4b-8708-f94cfeaf0777",
  "externalID": "test_merchant_id_1",
  "trackerID": "e42e0768-d913-4b4b-8708-f94cfeaf0777",
  "status": "COMPLETED",
  "amount": "1000",
  "commission": "100",
  "currency": "RUB",
  "bank": "Озон Банк (OZON)",
  "method": "CARD",
  "receiver": "2200154965960000",
  "holder": "Иванов Иван Иванович",
  "description": "Тестовая транзакция",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### Структура callback для PayOut:

```json
{
  "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
  "externalID": "test_merchant_id_2",
  "trackerId": "f5ef6b73-0952-4602-a306-82ef1f755f85",
  "status": "COMPLETED",
  "amount": "5000",
  "commission": "355",
  "currency": "RUB",
  "bank": "Озон Банк (OZON)",
  "method": "CARD",
  "receiver": "4000000000000000",
  "holder": "Иванов Иван Иванович",
  "description": "Тестовая транзакция",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### Параметры callback

| Параметр | Тип | Описание |
|----------|-----|----------|
| externalID | string | Ваш уникальный ID транзакции |
| trackerId | string | Опциональный ID для отслеживания (если передан) |
| status | string | Новый статус транзакции |
| amount | string | Сумма транзакции |
| commission | string | Комиссия |
| currency | string | Валюта |
| bank | string | Банк |
| method | string | Метод платежа/выплаты |
| receiver | string | Реквизиты получателя |
| holder | string | Имя держателя карты |
| description | string | Описание транзакции |
| timestamp | string | Время изменения статуса в формате ISO 8601 |

### Заголовки callback запроса

```http
Content-Type: application/json
User-Agent: Way2Pay-Callback/1.0
```

### Безопасность callback уведомлений

Для обеспечения безопасности рекомендуется:

1. **Проверка IP-адресов** - Ограничить доступ к callback URL только с IP-адресов Way2Pay
2. **HTTPS** - Использовать только защищенные HTTPS URL для callback
3. **Валидация данных** - Проверять корректность полученных данных
4. **Идемпотентность** - Обрабатывать возможные дублирующиеся уведомления
5. **Таймауты** - Отвечать на callback запросы в течение 30 секунд

### Обработка callback в коде

#### Пример для PHP:
```php
<?php
$input = file_get_contents('php://input');
$data = json_decode($input, true);

if ($data && isset($data['externalID'], $data['status'])) {
    // Обработка уведомления
    updateTransactionStatus($data['externalID'], $data['status']);
    
    // Возврат успешного ответа
    http_response_code(200);
    echo json_encode(['status' => 'success']);
} else {
    http_response_code(400);
    echo json_encode(['error' => 'Invalid data']);
}
?>
```

#### Пример для Node.js:
```javascript
app.post('/callback', express.json(), (req, res) => {
    const { externalID, status, amount, currency } = req.body;
    
    if (externalID && status) {
        // Обработка уведомления
        updateTransactionStatus(externalID, status);
        
        res.json({ status: 'success' });
    } else {
        res.status(400).json({ error: 'Invalid data' });
    }
});
 ```
 
 ### Обработка callback

1. **Ваш сервер должен отвечать HTTP 200** для подтверждения получения
2. **Время ожидания ответа**: 30 секунд
3. **Повторные попытки**: В случае ошибки система выполнит до 3 повторных попыток
4. **Безопасность**: Рекомендуется проверять IP-адрес отправителя

### Пример обработки callback (PHP)

```php
<?php
// Получаем данные callback
$input = file_get_contents('php://input');
$data = json_decode($input, true);

if ($data) {
    $externalID = $data['externalID'];
    $status = $data['status'];
    $amount = $data['amount'];
    
    // Обновляем статус транзакции в вашей системе
    updateTransactionStatus($externalID, $status);
    
    // Возвращаем успешный ответ
    http_response_code(200);
    echo 'OK';
} else {
    http_response_code(400);
    echo 'Invalid data';
}
?>
```

### Пример обработки callback (Node.js)

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/callback', (req, res) => {
    const { externalID, status, amount, currency, method, timestamp } = req.body;
    
    // Обновляем статус транзакции в вашей системе
    updateTransactionStatus(externalID, status);
    
    // Возвращаем успешный ответ
    res.status(200).send('OK');
});

app.listen(3000, () => {
    console.log('Callback server running on port 3000');
});
```

## Поддержка

Для получения технической поддержки и дополнительной информации:

- **Техническая поддержка**: https://t.me/Way2Pay_CTO
- **Статус системы**: https://api.way2pay.top/monitoring/health
- **Документация обновлена**: 08.08.2025

## Обновления от 08.08.2025

### Добавлены новые коды ошибок

| Код | Сообщение | Статус |
|-----|-----------|--------|
| 2004 | request timeout | 401 |
| 2005 | invalid Signature | 401 |
| 2007 | invalid NONCE | 401 |
| 60013 | commission doesnt exists | 400 |
| 60014 | bank doesnt exists | 400 |
| 60015 | method doesnt exists | 400 |

### Удаление ненужных полей с эндпоинта `/api/v1/banks`
```json
{
  "isActive": true,
  "isTestMode": false,
  "createdAt": "2025-07-26T12:41:35.442Z",
  "updatedAt": "2025-07-26T12:41:35.442Z"
}
```

---

*Данная документация регулярно обновляется. Следите за изменениями и новыми возможностями API.*
