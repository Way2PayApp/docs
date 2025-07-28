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

function generateSignature(secret, message) {
    return crypto.createHmac('sha512', secret)
                 .update(message)
                 .digest('hex');
}
```

#### PHP
```php
function generateSignature($secret, $message) {
    return hash_hmac('sha512', $message, $secret);
}
```

### Формирование Сообщения

Сообщение (message), используемое для генерации подписи, формируется по-разному в зависимости от метода HTTP запроса:

- **GET Запрос**: Сообщение составляется из URL и закодированных параметров запроса.
- **POST Запрос**: Сообщение включает URL и JSON тело запроса.

#### <span style="color:red">
*JSON ключи в теле (body) и query параметры запроса должны идти в алфавитном порядке!* </span>

#### Пример формирования сообщения для GET запроса:
- **URL**: `/api/v1/balance`
- **Expires**: `1721585422` - время жизни запроса в формате UNIX по UTC
- **Сообщение для подписи**: `/api/v1/balance1721585422`

#### Пример формирования сообщения для POST запроса:
- **URL**: `https://api.way2pay.top/api/v1/pay-in`
- **Expires**: `1721585422`
- **Тело запроса**: `{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}`
- **Сообщение для подписи**: `/api/v1/pay-in{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}1721585422`

### Обязательные заголовки

Каждый запрос к API должен включать следующие заголовки:

- **Content-Type**: `application/json`
- **Public-Key**: Ваш публичный ключ, предоставленный *Way2Pay*
- **Expires**: Время истечения жизни запроса (UNIX timestamp)
- **Signature**: Подпись HMAC-SHA512, сгенерированная с использованием вашего приватного ключа

#### Пример заголовков

```http
Content-Type: application/json
Expires: 1717025133
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
Host: your-domain.com
Content-Type: application/json
Expires: 1717025133
Public-Key: your_public_key_here
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

### POST Запрос

Пример POST запроса для создания PayIn:

```http
POST /api/v1/pay-in HTTP/1.1
Host: way2pay.top
Content-Type: application/json
Expires: 1717025134
Public-Key: your_public_key_here
Signature: 3336894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8678

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

Все ответы от API Payment SAAS возвращаются в формате JSON. Ответ включает в себя:

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
    "code": 60006
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
- `X-Environment`: sandbox | test | production (опционально, по умолчанию production)

#### Пример ответа
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "SBER",
      "displayName": "Сбербанк",
      "isActive": true,
      "supportedCurrencies": ["RUB"]
    },
    {
      "id": 2,
      "name": "TINKOFF",
      "displayName": "Тинькофф Банк",
      "isActive": true,
      "supportedCurrencies": ["RUB"]
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
      "code": "RUB",
      "name": "Российский рубль",
      "symbol": "₽",
      "isActive": true
    },
    {
      "id": 2,
      "code": "USD",
      "name": "Доллар США",
      "symbol": "$",
      "isActive": true
    }
  ]
}
```

### 4. Получение комиссий

**GET** `/api/v1/commissions`

Получение информации о комиссиях (доступно только для ADMIN и SUPER_ADMIN).

#### Заголовки
- `X-Environment`: sandbox | test | production (опционально)

#### Пример ответа
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "type": "PAY_IN",
      "method": "CARD",
      "currency": "RUB",
      "percentage": "2.5",
      "fixedAmount": "0",
      "minAmount": "10",
      "maxAmount": "0"
    }
  ]
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
| method | string | Да | Метод платежа: CARD, SBP, ACCOUNT, CROSSBORDER | "CARD" |

#### Пример запроса
```json
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

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_merchant_id_2",
    "amount": "1000",
    "currency": "RUB",
    "status": "created",
    "method": "CARD",
    "bank": "ALFA",
    "receiver": "2200154965960000",
    "holder": "Иванов Иван Иванович",
    "cardNumber": "2200154965960000",
    "phoneNumber": "+79001234567",
    "nspkURL": "https://payment.example.com/pay/uuid-here",
    "country": "RU",
    "createdAt": "2024-01-01T12:00:00Z"
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
| bank | string | Да | Код банка | "SBER" |
| method | string | Да | Метод выплаты: card, sbp, account | "card" |
| currencyId | string | Да | Код валюты | "RUB" |
| callbackURL | string | Да | URL для получения callback | "https://example.com/callback" |
| amount | string | Да | Сумма выплаты | "5000" |
| receiver | string | Да | Реквизиты получателя | "4000000000000000" |
| holder | string | Да | Имя получателя | "Иванов Иван Иванович" |

#### Пример запроса
```json
{
  "externalID": "test_payout_123",
  "bank": "SBER",
  "method": "card",
  "currencyId": "RUB",
  "callbackURL": "https://example.com/callbacks/payout",
  "amount": "5000",
  "receiver": "4000000000000000",
  "holder": "Иванов Иван Иванович"
}
```

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_payout_123",
    "amount": "5000",
    "currency": "RUB",
    "status": "created",
    "method": "card",
    "receiver": "4000000000000000",
    "holder": "Иванов Иван Иванович",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

### 7. Получение заявки по ID

**GET** `/api/v1/pay-in/{id}` или `/api/v1/pay-out/{id}`

Получение информации о конкретной заявке.

#### Параметры URL
- `id` - ID заявки

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_merchant_id_2",
    "amount": "1000",
    "currency": "RUB",
    "status": "completed",
    "method": "CARD",
    "createdAt": "2024-01-01T12:00:00Z",
    "updatedAt": "2024-01-01T12:05:00Z"
  }
}
```

## Статусы транзакций

### PayIn статусы
- `created` - Заявка создана
- `pending` - Ожидает оплаты
- `completed` - Успешно завершена
- `failed` - Ошибка при обработке
- `canceled` - Отменена

### PayOut статусы
- `created` - Заявка создана
- `pending` - В обработке
- `completed` - Успешно завершена
- `failed` - Ошибка при обработке
- `canceled` - Отменена

## Обработка Ошибок

В случае ошибки ответ будет включать:

- **message**: Описание ошибки
- **code**: Код ошибки

### Коды ошибок

| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 10000 | unauthorized | 401 |
| 20000 | wrong input | 400 |
| 20001 | can't bind body to request model | 422 |
| 20002 | can't bind query parameters | 422 |
| 20003 | failed to parse key | 422 |
| 20004 | signature header value missing or malformed | 400 |
| 20005 | public-Key header value missing or malformed | 400 |
| 20006 | timestamp header value missing or outdated | 400 |
| 20012 | invalid query params | 400 |
| 20015 | conflict | 409 |
| 20016 | empty external ID | 400 |
| 30000 | forbidden | 403 |
| 30001 | no access to requested session | 403 |
| 30002 | requested sessions has expired | 403 |
| 30003 | user doesn't exists | 403 |
| 30004 | zero balance | 403 |
| 30005 | not enough balance | 402 |
| 30006 | amount less than min | 400 |
| 30007 | amount greater than max | 400 |
| 40000 | internal error | 500 |
| 60003 | empty Public-Key | 401 |
| 60004 | empty Expires | 401 |
| 60005 | empty Signature | 401 |
| 60006 | invalid Signature | 401 |
| 60007 | request timeout | 408 |
| 60008 | invalid Public-Key | 400 |
| 60009 | empty external ID | 400 |
| 60010 | external ID already exists | 409 |
| 60011 | payment doesn't exists | 404 |
| 60012 | payment is finalized | 409 |

## Тестовые окружения

Система поддерживает работу с различными окружениями через заголовок `X-Environment`:

- **sandbox** - песочница для тестирования
- **test** - тестовое окружение
- **production** - продакшн окружение (по умолчанию)

### Использование

Добавьте заголовок в ваши запросы:
```http
X-Environment: sandbox
```

### Поддерживаемые модули

- CommissionsModule
- PayInModule
- PayOutModule
- BanksModule
- BalanceModule
- ProcessingModule
- MerchantsModule
- TransactionsService
- CommissionsService

## Callback уведомления

Система автоматически отправляет HTTP POST уведомления на указанный `callbackURL` при изменении статуса транзакции.

### Структура callback для PayIn

```json
{
  "externalID": "test_merchant_id_2",
  "status": "completed",
  "amount": "1000",
  "currency": 1,
  "method": "CARD",
  "timestamp": "2024-01-15T12:00:00.000Z",
  "trackerId": "optional_tracker_id"
}
```

### Структура callback для PayOut

```json
{
  "externalID": "test_payout_123",
  "status": "completed",
  "amount": "5000",
  "currency": 1,
  "method": "card",
  "receiver": "4000000000000000",
  "timestamp": "2024-01-15T12:00:00.000Z",
  "trackerId": "optional_tracker_id"
}
```

### Параметры callback

| Параметр | Тип | Описание |
|----------|-----|----------|
| externalID | string | Ваш уникальный ID транзакции |
| status | string | Новый статус транзакции |
| amount | string | Сумма транзакции |
| currency | number | ID валюты |
| method | string | Метод платежа/выплаты |
| receiver | string | Реквизиты получателя (только для PayOut) |
| timestamp | string | Время изменения статуса в формате ISO 8601 |
| trackerId | string | Опциональный ID для отслеживания (если передан) |

### Заголовки callback запроса

```http
Content-Type: application/json
User-Agent: Way2Pay-Callback/1.0
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

- **Техническая поддержка**: https://way2pay.top/#contact
- **Статус системы**: https://status.way2pay.top
- **Документация обновлена**: 2024-01-15

---

*Данная документация регулярно обновляется. Следите за изменениями и новыми возможностями API.*
