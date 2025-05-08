# System Architecture And Flow Document: Real-Time Inventory Check Component

## System Overview

This component addresses the "phantom inventory" problem by providing accurate and up-to-date stock levels to customers and managing inventory state throughout the purchase lifecycle.

The system leverages a serverless architecture on AWS, consisting of:

- **AWS Lambda**: Hosts the core business logic for checking, updating, reserving, and releasing inventory. Also runs a scheduled function for low-stock notifications.
- **Amazon API Gateway**: Provides RESTful API endpoints for the frontend and administrative clients to interact with the Lambda functions securely.
- **Amazon DynamoDB**: Acts as the persistent, highly available database for storing product inventory data, including stock levels, reservations, thresholds, and status. Utilizes strong consistency for critical operations.
- **Amazon S3**: Hosts the static frontend assets (HTML, CSS, JavaScript) for the customer-facing components and admin panel.
- **Amazon SNS**: Publishes notifications when product inventory levels fall below a configured threshold.
- **Amazon CloudWatch**: Collects logs, metrics, and triggers scheduled events for the Lambda functions.

The system ensures data consistency through conditional updates in DynamoDB and manages temporary stock allocations using DynamoDB's TTL feature.

## User Personas

### Persona: Alice, The Savvy Shopper

**Background:**
- Works full-time, shops online frequently for convenience
- Often compares prices and availability across sites

**Goals:**
- Find products quickly
- Confirm availability before adding to cart and checking out
- Complete purchases without hassle

**Pain Points:**
- Frustrated by ordering an item only to receive an email later saying it's out of stock
- Wasted time and effort
- Distrusts sites with poor inventory accuracy

**Needs from this Product:**
- Clear, accurate stock status on the product page
- Confidence that items added to the cart are likely available for purchase within a reasonable timeframe

### Persona: Mark, The Inventory Manager

**Background:**
- Responsible for managing stock levels across multiple product categories and potentially warehouses
- Deals with suppliers, incoming shipments, and order fulfillment issues

**Goals:**
- Ensure accurate stock counts in the system
- Update inventory levels efficiently
- Identify low-stock items before they run out
- Minimize manual checks and discrepancies

**Pain Points:**
- Difficulty knowing true stock levels in real-time
- Doesn't get alerted to low stock until it's too late
- Manual updates are time-consuming and error-prone
- Dealing with customer complaints from phantom inventory

**Needs from this Product:**
- Secure APIs to view and update stock
- Automated alerts for low stock
- Accurate data reflecting reservations

## User Workflows

### 1.1. Viewing Product Inventory

A customer browses the website and lands on a product details page, requiring the current stock status to be displayed.

1. Customer navigates to a product page on the e-commerce website
2. The frontend (served from S3) loads and executes JavaScript
3. The frontend makes an asynchronous GET request to `/inventory/{productId}` endpoint via API Gateway
4. API Gateway receives the request and routes it to the `getInventoryLevel` Lambda function
5. The `getInventoryLevel` Lambda queries the `inventory-management` DynamoDB table using `product_id` and `warehouse_id` (if applicable)
6. DynamoDB returns the current `stock_count`, `status`, and potentially `reserved_count` to the Lambda function
7. The Lambda function processes the data and formats a response
8. API Gateway receives the response from Lambda and returns it to the frontend
9. The frontend updates the product page to display the real-time inventory status

### 1.2. Adding Product to Cart (Inventory Reservation)

When a customer decides to purchase, the system temporarily reserves the items to prevent over-selling.

1. Customer clicks "Add to Cart" on a product page
2. The frontend sends a POST request to `/inventory/{productId}/reserve` via API Gateway, including the quantity to reserve
3. API Gateway routes the request to the `reserveInventory` Lambda function
4. The `reserveInventory` Lambda reads the current `stock_count` and `reserved_count` from DynamoDB
5. The Lambda checks if `stock_count - reserved_count` is greater than or equal to the requested quantity
6. If sufficient stock is available, the Lambda performs a conditional update on the DynamoDB item to increment `reserved_count` by the requested quantity
7. The update also sets or updates a `reservation_expiry` attribute with a TTL
8. DynamoDB confirms the update or rejects it if the condition fails
9. The Lambda function returns a success response (including a reservation ID) or an error response
10. API Gateway returns the response to the frontend
11. The frontend updates the cart status or displays an error message

### 1.3. Completing Checkout (Inventory Decrement)

Upon successful checkout, the reserved inventory is permanently decremented from the total stock.

1. Customer confirms the order during the checkout process
2. The frontend sends a PUT request to `/inventory/{productId}` via API Gateway
3. API Gateway routes the request to the `updateInventoryLevel` Lambda function
4. The Lambda performs a conditional update on the DynamoDB item:
    - Decrement `stock_count` by the reserved quantity
    - Decrement `reserved_count` by the same quantity
5. This update uses a conditional expression to ensure the decrement happens only if the reservation still exists
6. DynamoDB confirms the update or rejects it
7. The Lambda function returns a success or failure response
8. API Gateway returns the response to the backend order processing system or frontend

### 1.4. Admin Updating Inventory

An administrator adjusts stock levels, typically after receiving new shipments or correcting errors.

1. Admin user accesses the admin panel (served from S3)
2. Admin user authenticates
3. Admin user updates the stock quantity for a product in the admin UI
4. The frontend sends a PUT request to `/inventory/{productId}` via API Gateway with the new stock level
5. API Gateway routes the request to the `updateInventoryLevel` Lambda function
6. The `updateInventoryLevel` Lambda performs an update on the DynamoDB item
7. The Lambda also calculates the new status based on the new `stock_count` and `threshold_level`
8. DynamoDB confirms the update
9. The Lambda function returns a success or failure response
10. API Gateway returns the response to the admin panel

## Technical Flow

### 3.1. Inventory Read Flow

Data related to current inventory levels flows from the database to the user interface.

1. **Request Initiation:** Frontend sends GET request (`/inventory/{productId}`) to API Gateway with `product_id`
2. **Routing:** API Gateway extracts path parameters (`productId`) and passes them to `getInventoryLevel` Lambda
3. **Database Query:** `getInventoryLevel` Lambda uses `product_id` to perform a GetItem operation on the `inventory-management` DynamoDB table. Specifies `ConsistentRead: true`
4. **Data Retrieval:** DynamoDB retrieves the item with the matching composite key and returns attributes like `stock_count`, `reserved_count`, `threshold_level`, `status`, `last_updated`
5. **Data Processing:** Lambda receives the data, potentially formats the status message based on `stock_count` and `threshold_level`
6. **Response Transmission:** Lambda returns a JSON response containing the relevant inventory data (e.g., `{ "stock": 10, "status": "IN_STOCK" }`) to API Gateway
7. **API Response:** API Gateway serializes the JSON response and returns it as the HTTP response body to the Frontend
8. **UI Update:** Frontend JavaScript parses the JSON and updates the displayed stock information

### 3.2. Inventory Update Flow

Data representing changes to stock levels flows from an administrative source or the checkout process to the database.

1. **Update Initiation:** Admin panel sends PUT request (`/inventory/{productId}`) to API Gateway with `product_id` and update payload
2. **Routing:** API Gateway routes the request body and path parameters to `updateInventoryLevel` Lambda
3. **Database Update:** `updateInventoryLevel` Lambda parses the payload and performs an UpdateItem operation on the `inventory-management` DynamoDB table
4. **Data Persistence:** DynamoDB applies the update if conditions are met, ensuring data integrity
5. **Response:** DynamoDB returns confirmation of the update or an error to the Lambda
6. **Lambda Response:** Lambda returns a success or failure status
7. **API Response:** API Gateway returns the HTTP response to the Admin panel/Checkout process

### 3.3. Low Stock Notification Flow

Data indicating low stock levels flows from the database, through a scheduled process, to the notification service.

1. **Scheduled Trigger:** A CloudWatch Event Rule triggers the `checkThresholds` Lambda function on a schedule
2. **Database Scan/Query:** `checkThresholds` Lambda performs a scan or query on the `inventory-management` DynamoDB table to find items where `stock_count <= threshold_level` AND `status != "LOW_STOCK"`
3. **Identify Low Stock:** Lambda processes the results, identifying products that meet the low stock criteria
4. **Update Status:** For identified items, the Lambda updates their `status` attribute in DynamoDB to "LOW_STOCK"
5. **Publish Notification:** For each newly identified low-stock item, the Lambda publishes a message to the `inventory-alerts` SNS topic
6. **Notification Delivery:** SNS delivers the message to subscribed endpoints (e.g., email addresses configured for admin alerts)

## AWS Architecture Diagram

The component uses a serverless architecture on AWS with Lambda functions for business logic, API Gateway for RESTful endpoints, DynamoDB for persistent storage, S3 for hosting frontend assets, SNS for notifications, and CloudWatch for monitoring and scheduled events.mbda functions for business logic, API Gateway for RESTful endpoints, DynamoDB for persistent storage, S3 for hosting frontend assets, SNS for notifications, and CloudWatch for monitoring and scheduled events.