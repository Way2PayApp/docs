# API Документация Way2Pay

## Введение


Добро пожаловать в документацию по API **Way2Pay**. Наш API предоставляет вам мощные возможности для программного взаимодействия с нашими сервисами, обеспечивая простую и надёжную интеграцию с вашими приложениями.

## Аутентификация и Подпись

Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью секретного ключа (SecretKey), который мы предоставляем нашим клиентам. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи

Подпись генерируется с использованием алгоритма HMAC-SHA256. Ниже приведен пример функции генерирования подписи (Signature):

#### JavaScript
```javascript
const crypto = require('crypto');

function generateSignature(secretKey, message) {
  return crypto
    .createHmac('sha256', secretKey)
    .update(message)
    .digest('hex');
}
```

### Формирование Сообщения

Сообщение (message), используемое для генерации подписи, формируется следующим образом:

1. Преобразуйте тело запроса в JSON-строку
2. Добавьте к этой строке значение nonce (временная метка)
3. Полученная строка используется для генерации подписи

#### Пример кода на JavaScript:
```javascript
const data = { amount: "1000", currency: "RUB" }; // Тело запроса
const nonce = Math.floor(Date.now() / 1000) + 300; // Текущее время + 5 минут
const dataString = JSON.stringify(data);
const message = `${dataString}${nonce}`;
const signature = generateSignature(secretKey, message);
```


### Примеры

Для POST запроса на `https://way2pay.app/api/v1/pay-in` с телом:
```json
{
  "externalID": "test123",
  "currency": "RUB",
  "callbackURL": "https://test.com/callback",
  "description": "Тестовый платеж",
  "amount": "1000"
}
```

1. Nonce: 1721585422 - время жизни запроса в формате UNIX по UTC, рекомендуем добавлять к текущему времени 5 минут
2. Сообщение для подписи: {"externalID":"test123","currency":"RUB","callbackURL":"https://test.com/callback","description":"Тестовый платеж","amount":"1000"}1721585422

### Заголовки Запросов

Каждый запрос к API должен включать следующие заголовки:

1. Content-Type: application/json
2. Public-Key: Ваш публичный ключ, предоставленный Way2Pay.
3. Nonce: Временная метка в формате UNIX, которая должна быть больше, чем в предыдущем запросе.
4. Signature: Подпись HMAC-SHA256, сгенерированная с использованием вашего секретного ключа и сообщения.

### Пример заголовков
```javascript
Content-Type: application/json
Nonce: 1721585422
Public-Key: your_public_key
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b
```

### Выполнение Запроса

Ниже приведен пример того, как выполнить POST запрос к API Way2Pay:

#### POST Запрос
```javascript
POST /api/v1/pay-in HTTP/1.1
Host: api.way2pay.com
Content-Type: application/json
Nonce: 1721585422
Public-Key: your_public_key
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b

{
  "externalID": "test123",
  "currency": "RUB",
  "callbackURL": "https://test.com/callback",
  "description": "Тестовый платеж",
  "amount": "1000"
}
```

#### Ответ

Все ответы от API Way2Pay будут в формате JSON. Ответ включает в себя:

1. success - Статус запроса (true/false).
2. data - Данные, возвращенные API (только в случае успешного ответа).
3. error - Данные об ошибке (только в случае ошибочного ответа).

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "id": "6f3c9a2d-7b5e-4f8a-9d3b-1c2e4f5a6b7c",
    "externalID": "test123",
    "currency": "RUB",
    "amount": "1000",
    "status": "PENDING",
    "method": "CARD",
    "bank": "SBER",
    "cardNumber": "2200123456789012",
    "holder": "Иванов Иван",
    "createdAt": "2025-07-06T16:30:22.123Z",
    "updatedAt": "2025-07-06T16:30:22.123Z"
  }
}
```

#### Обработка Ошибок

В случае ошибки ответ будет включать:

- **success**: false
- **error**: Объект с информацией об ошибке
    **message**: Описание ошибки
    **code**: Код ошибки

#### Пример ответа с ошибкой
```json
{
  "success": false,
  "error": {
    "message": "Invalid Signature",
    "code": 60006
  }
}
```

## Коды ошибок

| Код ошибки | Сообщение                               | HTTP статус код |
|------------|----------------------------------------|------------------|
| 10000      | unauthorized                           | 401              |
| 20000      | wrong input                            | 400              |
| 20001      | can't bind body to request model       | 422              |
| 20002      | can't bind query parameters            | 422              |
| 20003      | failed to parse key                    | 422              |
| 20004      | signature header value missing or malformed | 400           |
| 20005      | public-Key header value missing or malformed | 400         |
| 20006      | nonce header value missing or outdated | 400              |
| 30000      | forbidden                              | 403              |
| 30001      | user doesn't exists	                  | 403              |
| 30003      | user doesn't exists                    | 403              |
| 30004      | zero balance                           | 403              |
| 30005      | not enough balance                     | 402              |
| 30006      | amount less than min                   | 400              |
| 30007      | amount greater than max                | 400              |
| 40000      | internal error                         | 500              |
| 60000      | ticker doesnt exists                   | 400              |
| 60001      | ticker type doesnt exists              | 400              |
| 60002      | invalid status                         | 400              |
| 60003      | empty Public-Key                       | 401              |
| 60004      | empty Nonce                            | 401              |
| 60005      | empty Signature                        | 401              |
| 60006      | invalid Signature                      | 401              |
| 60007      | request timeout                        | 408              |
| 60008      | invalid Public-Key                     | 400              |
| 60009      | empty externalID                       | 400              |
| 60010      | externalID already exists              | 409              |
| 60011      | payment doesn't exists                 | 404              |
| 60012      | payment is finalized                   | 409              |

