# Специфікація взаємодії та динаміки системних сценаріїв (Завдання 3)

## 1. Обраний Use Case
**UC-02: Checkout Order (Оформлення замовлення)** — прецедент використання, розроблений у Завданні 2.

---

## 2. Учасники взаємодії (Participants)

### Актори та зовнішні сервіси:
- `RegisteredCustomer` — користувач системи, який ініціює оформлення та оплату замовлення.
- `PaymentGateway` — зовнішній сервіс проведення платіжних транзакцій.

### Внутрішні компоненти системи:
- `WebUI` — клієнтський інтерфейс системи.
- `OrderController` — контроллер обробки запитів замовлення.
- `StockService` — сервіс перевірки та резервування складських залишків.
- `PaymentService` — сервіс управління платіжними сесіями.
- `SystemScheduler` — фоновий планувальник для контролю таймаутів замовлень.
- `Database` — сховище даних сутностей системи.

### Сутності системи (з ER-моделі Завдання 1):
- `PRODUCT_ITEM` — сутність товару та його залишку `stock_quantity`.
- `ORDER` — сутність замовлення та його стану (статуси `Pending_Payment`, `Cancelled_Expired`, `Processing`).
- `ORDER_ITEM` — позиції замовлення з фіксованою ціною `unit_price`.
- `PAYMENT` — сутність платіжної транзакції.

---

## 3. Опис покрокових сценаріїв взаємодії

### Сценарій 1: Успішне створення замовлення та резервування залишків (Happy Path - `scenario1.mmd`)
1. Покупець передає деталі кошика та спосіб оплати через `WebUI` (`submitOrder(cartItems, paymentMethod)`).
2. `WebUI` викликає метод `submitOrder(customerId, items, paymentMethod)` у `OrderController`.
3. `OrderController` звертається до `StockService` через `checkAndReserveStock(items)`.
4. У циклі для кожної позиції `StockService` валідує `stock_quantity` сутності `PRODUCT_ITEM`, зменшує залишок та повертає статус `stockReserved = true`.
5. `OrderController` створює екземпляр `ORDER` у статусі `Pending_Payment` та фіксує позиції `ORDER_ITEM` із ціною `unit_price`.
6. Якщо обрано онлайн-оплату (`opt`), `OrderController` викликає `processPayment(orderId, amount)` у `PaymentService`, який проводить транзакцію через `PaymentGateway` (`executeTransaction(cardDetails, amount)`).
7. Після підтвердження (`Success`) `PaymentService` створює запис `PAYMENT` зі статусом `Success` та оновлює статус `ORDER` на `Processing`.
8. `OrderController` повертає `200 OK (orderId, status='Processing')` у `WebUI`, а покупець бачить екран підтвердження успішного замовлення.


### Сценарій 2: Відсутність товару на складі (Error Path — REQ-02, `scenario2.mmd`)
1. Покупець ініціює оформлення замовлення через `WebUI` (`submitOrder(cartItems, paymentMethod)`).
2. `OrderController` викликає `checkAndReserveStock(items)` у `StockService`.
3. `StockService` перевіряє `stock_quantity` сутності `PRODUCT_ITEM` у `Database` і виявляє нестачу товару (`stock_quantity < requested_qty`).
4. `StockService` повертає помилку `InsufficientStockException`.
5. `OrderController` відхиляє створення замовлення та надсилає `400 Bad Request` у `WebUI`.
6. `WebUI` відображає повідомлення про відсутність товару.

### Сценарій 3: Скасування замовлення за таймаутом (Timeout Path — REQ-05, `scenario3.mmd`)
1. Фоновий планувальник `SystemScheduler` кожні 5 хвилин (`loop`) викликає `checkExpiredOrders()` у `OrderController`.
2. `OrderController` шукає в `Database` записи `ORDER` зі статусом `Pending_Payment`, створені понад 30 хвилин тому.
3. Якщо знайдено прострочені замовлення (`opt`), `OrderController` змінює їх статус `ORDER` на `Cancelled_Expired`.
4. `OrderController` викликає `releaseStock(items)` у `StockService`.
5. `StockService` повертає зарезервовану кількість товару до `stock_quantity` сутності `PRODUCT_ITEM`.

### Сценарій 4: Неуспішна спроба онлайн-платежу (Error Path — REQ-04, `scenario4.mmd`)
1. Покупець обирає онлайн-оплату замовлення у `WebUI` (`payOrder(orderId, cardDetails)`).
2. `OrderController` викликає `processPayment(orderId, amount)` у `PaymentService`.
3. `PaymentService` ініціює транзакцію `executeTransaction(cardDetails, amount)` у `PaymentGateway`.
4. `PaymentGateway` повертає статус відхилення `PaymentRejected`.
5. `PaymentService` створює запис у сутності `PAYMENT` зі статусом `Failed`.
6. `OrderController` залишає `ORDER` у статусі `Pending_Payment` та повертає у `WebUI` повідомлення про помилку оплати з пропозицією повторити спробу.
