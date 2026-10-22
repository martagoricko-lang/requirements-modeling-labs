# Специфікація динаміки взаємодії (Завдання 3)

## 1. Обраний прецедент використання (Use Case)
**UC-02: Checkout Order (Оформлення замовлення)** — основний бізнес-сценарій платформи e-Commerce, який описує процес створення замовлення, перевірку складських залишків варіації товару (SKU), ініціалізацію оплати через платіжний шлюз та надсилання сповіщення клієнту.

---

## 2. Учасники взаємодії (Participants)

### Актори та зовнішні сервіси:
- **Customer (Покупець):** Користувач, який підтверджує оформлення кошика.
- **Payment Gateway (Платіжний шлюз):** Зовнішній сервіс проведення онлайн-платежів.
- **Notification Service (Сервіс сповіщень):** Зовнішня служба відправки Email/SMS.

### Внутрішні компоненти / сутності системи (з ER-моделі Завдання 1):
- **Web UI (Frontend):** Інтерфейс користувача.
- **Order Service (Backend API):** Основна бізнес-логіка обробки замовлень.
- **Database (БД):** Сховище даних (`ORDER`, `ORDER_ITEM`, `PRODUCT_ITEM`, `PAYMENT`).

---

## 3. Покроковий сценарій взаємодії (Main & Alternative Flows)

1. **Покупець** надсилає запит `submitOrder(cartItems, paymentMethod)` через Web UI.
2. **Web UI** викликає `createOrder(customerId, items)` у **Order Service**.
3. **Order Service** перевіряє залишки товарів `validateStock(productItemId, quantity)` у **Database**.
   - **Умовна гілка (`alt`):**
     - **[Залишків достатньо]:** Database повертає `stockAvailable = true`.
     - **[Залишків недостатньо]:** Database повертає `stockAvailable = false`. Order Service кидає помилку `400 Out of Stock`, Web UI відображає сповіщення Покупцю.
4. Order Service створює запис замовлення `saveOrder(status="Pending_Payment")` у **Database** та резервує складські залишки `reserveStock(quantity)`.
5. **Опціональний блок (`opt`) — Онлайн-оплата:**
   - Якщо обрано онлайн-оплату, Order Service надсилає запит `initiatePayment(orderId, amount)` до **Payment Gateway**.
   - **Payment Gateway** обробляє транзакцію `processTransaction()` та повертає статус `paymentStatus = Success / Failed`.
   - При успішній оплаті Order Service оновлює статус замовлення `updateOrderStatus("Processing")` та створює запис у таблиці `PAYMENT`.
6. Order Service асинхронно викликає **Notification Service** для відправки підтвердження `sendOrderConfirmation(email, orderDetails)`.
7. Order Service повертає успішну відповідь `201 Created (orderId)` на Web UI, який показує Покупцю екран підтвердження замовлення.
