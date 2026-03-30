# cloudKitchen

## 📌 Objective
This is a Ruby on Rails backend service designed to process home-cooked meal orders. It focuses on high-reliability in distributed systems, specifically addressing real-world edge cases such as duplicate requests, race conditions, and downstream API failures. 

This project was built to satisfy the requirements of the AI-Assisted Build assignment.

## 🏗 System Architecture & Edge Case Handling

This service models a strict inventory system where meals have limited portions. Here is how the core requirements are addressed:

### 1. Idempotency & Duplicate Request Handling
* **Mechanism:** The API requires an `idempotency_key` (UUID) for every `POST /api/v1/orders` request. 
* **Database Constraint:** A unique index on `idempotency_key` in the `order_requests` table ensures that even under heavy load, duplicate records cannot be created at the database level.
* **Application Logic:** The controller uses `find_or_initialize_by`. If a user clicks "Order" multiple times, the subsequent requests return the existing order's status without re-enqueueing the background job.

### 2. Concurrency & Race Conditions
* **Scenario:** Multiple users attempt to order the last available portion of a meal at the exact same millisecond.
* **Mechanism:** Pessimistic Database Locking.
* **Implementation:** Inside the ActiveJob worker, `Meal.transaction` and `meal.lock!` are used. This forces the database to serialize updates to the `available_portions` column, guaranteeing the inventory never drops below zero.

### 3. Background Processing & Retries
* **Mechanism:** `ProcessOrderJob` handles the heavy lifting asynchronously.
* **Downstream Failures:** The job simulates a call to an external "Kitchen Gateway API" which has a random chance of failing.
* **Safe Retries:** If the downstream API fails, the job retries up to 3 times. Because the database lock and inventory deduction happen *after* the external call (or are rolled back if an error is raised within the transaction), retrying the job never results in a duplicate inventory deduction.

### 4. Cancellation Handling
* **Mechanism:** A `POST /api/v1/orders/:id/cancel` endpoint allows users to cancel pending orders.
* **Job Intelligence:** When `ProcessOrderJob` picks up a task, its very first check is `return if order.cancelled?`. This prevents the system from processing orders that were cancelled while sitting in the queue.

---

## 🚀 Setup & Installation

1. **Clone the repository**
2. **Install dependencies:**
   ```bash
   bundle install
3. Database Setup: This will run the migrations and populate the database with test scenarios (high inventory, low inventory, and out-of-stock items).
**rails db:create db:migrate db:seed**
4. Start the server:
**rails s**

Testing the API (cURL Examples)
1. Create a New Order (Standard Flow)
curl -X POST http://localhost:3000/api/v1/orders \
-H "Content-Type: application/json" \
-d '{
  "order": {
    "idempotency_key": "uuid-1234-5678",
    "meal_id": 1,
    "quantity": 2
  }
}'
2. Test Idempotency (Duplicate Request)
Run the exact same command from Step 1 again.
Expected Result: You will receive a 200 OK (instead of 202 Accepted) with a message stating "Request already received". The database remains unchanged.

3. Test Cancellation
Cancel an order using its idempotency key.
curl -X POST http://localhost:3000/api/v1/orders/uuid-1234-5678/cancel \
-H "Content-Type: application/json"

**AI Assistance Disclosure**
As per the assignment guidelines, AI tools (specifically Gemini and cursor) were utilized effectively during the development of this project.
