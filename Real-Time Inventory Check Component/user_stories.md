## 1. Executive Summary

The Real-Time Inventory Check Component is aimed at solving the persistent problem of "phantom inventory." By providing customers with accurate, up-to-the-minute stock information directly on product pages and managing inventory reservations during the shopping process, this component will significantly reduce order cancellations due to lack of stock, improve customer satisfaction, and enable more efficient inventory management for administrators. Leveraging a serverless AWS architecture (Lambda, API Gateway, DynamoDB, SNS), this system will be scalable, reliable, and cost-effective.

## 2. Component Vision

The vision for the Real-Time Inventory Check Component is to establish a foundation of trust and reliability in our e-commerce platform's inventory data.

*   **Purpose:** To display current stock levels to customers in real-time and provide administrators with the ability to manage inventory accurately and receive timely low-stock alerts.
*   **Users:**
    *   **E-commerce Customers:** Seeking to purchase products and need reliable information on availability before ordering.
    *   **E-commerce Administrators / Merchandisers:** Responsible for managing stock levels, fulfilling orders, and needing accurate data and alerts to prevent stockouts or over-selling.
*   **Business Goals:**
    *   Increase customer satisfaction by reducing order cancellations due to unavailable stock.
    *   Decrease operational costs associated with processing and canceling orders with phantom inventory.
    *   Improve inventory turnover and reduce holding costs by enabling better stock management and forecasting based on 
    real-time data.
    *   Provide administrators with effective tools and alerts for proactive inventory management.

## 3. User Personas

**Persona: Alice, The Savvy Shopper**

*   **Background:** Works full-time, shops online frequently for convenience. Often compares prices and availability across sites.
*   **Goals:** Find products quickly, confirm availability *before* adding to cart and checking out, complete purchases without hassle.
*   **Pain Points:** Frustrated by ordering an item only to receive an email later saying it's out of stock. Wasted time and effort. Distrusts sites with poor inventory accuracy.
*   **Needs from this Product:** Clear, accurate stock status on the product page. Confidence that items added to the cart are likely available for purchase within a reasonable timeframe.

**Persona: Mark, The Inventory Manager**

*   **Background:** Responsible for managing stock levels across multiple product categories and potentially warehouses. Deals with suppliers, incoming shipments, and order fulfillment issues.
*   **Goals:** Ensure accurate stock counts in the system, update inventory levels efficiently, identify low-stock items before they run out, minimize manual checks and discrepancies.
*   **Pain Points:** Difficulty knowing true stock levels in real-time. Doesn't get alerted to low stock until it's too late. Manual updates are time-consuming and error-prone. Dealing with customer complaints from phantom inventory.
*   **Needs from this Product:** Secure APIs to view and update stock. Automated alerts for low stock. Accurate data reflecting reservations.

## 4. Feature Specifications

### Feature 4.1: Real-Time Stock Display

*   **User Stories:**
    *   As Alice, I want to see the current stock level for a product on the product page so I know if it's available before attempting to buy.
    *   As Alice, I want the stock level information to be accurate and updated frequently so I can trust what I see.
*   **Acceptance Criteria:**
    *   The product page must display the current `stock_count` (minus `reserved_count`) for the specific `product_id` and relevant `warehouse_id` (if applicable).
    *   The stock information must be fetched using the `GET /inventory/{productId}` API endpoint.
    *   The displayed stock information must be strongly consistent with the data in DynamoDB.
    *   The stock status (`IN_STOCK`, `LOW_STOCK`, `OUT_OF_STOCK`) derived from `stock_count` and `threshold_level` should be displayed visually (e.g., text like "In Stock", "Low Stock (5 left)", "Out of Stock").
    *   If `stock_count` is 0, the display must clearly indicate "Out of Stock" and ideally disable the "Add to Cart" button.
    *   If `stock_count` is <= `threshold_level` (and > 0), the display must indicate "Low Stock" and show the remaining quantity (e.g., "Only 3 left!").
    *   The inventory check must complete and display within 500ms under normal network conditions.
*   **Edge Cases:**
    *   API call fails: Display a friendly error message like "Stock status unavailable" and potentially disable "Add to Cart" as a safety measure.
    *   Product not found in inventory system: Display "Stock status unavailable" or assume out of stock.
    *   Network latency: Implement a loading indicator while fetching stock data.

### Feature 4.2: Secure Inventory Updates (Admin)

*   **User Stories:**
    *   As Mark, I want to manually update the stock level for a specific product/warehouse through a secure interface.
    *   As Mark, I want to ensure that my update is based on the latest data to prevent overwriting recent changes by another admin.
    *   As Mark, I want to be able to set or change the low-stock threshold for a product.
*   **Acceptance Criteria:**
    *   An API endpoint `PUT /inventory/{productId}` must exist to update inventory details (`stock_count`, `threshold_level`).
    *   This endpoint must be secured, requiring authentication (e.g., API Key as specified, or later integrate with Cognito/IAM).
    *   The update operation must use DynamoDB conditional expressions based on a version attribute or timestamp (`last_updated`) to prevent concurrent update conflicts. If a conflict occurs, the API should return an error (e.g., HTTP 409 Conflict).
    *   The `last_updated` attribute in DynamoDB must be automatically updated to the current ISO8601 timestamp on every successful modification.
    *   The API should return the updated inventory record upon success.
    *   Input validation must be performed for `stock_count` (must be non-negative integer) and `threshold_level` (must be non-negative integer).
*   **Edge Cases:**
    *   Invalid input data (non-numeric stock, negative values): API should return a 400 Bad Request error.
    *   Attempting to update a non-existent product: API should return a 404 Not Found error.
    *   Concurrent updates by multiple admins: The conditional expression must catch this, and one update should fail gracefully, prompting the user to refresh and try again.
    *   API key is missing or invalid: API should return a 401 Unauthorized or 403 Forbidden error.

### Feature 4.3: Inventory Reservation (Add to Cart & Checkout)

*   **User Stories:**
    *   As Alice, when I add an item to my cart, I want the system to temporarily hold that stock for me so it's likely available when I check out.
    *   As Alice, if I remove an item from my cart, I want that stock to become available for others.
    *   As Alice, when I successfully complete an order, I want the stock I purchased to be permanently removed from inventory.
    *   As Alice, if I abandon my cart, I want the reserved stock to eventually be released so others can buy it.
*   **Acceptance Criteria:**
    *   An API endpoint `POST /inventory/{productId}/reserve` must exist to reserve inventory.
    *   This API must take a requested `quantity` and a unique `reservationId` (e.g., cart item ID or session ID + product ID).
    *   The API must check if `stock_count` - `reserved_count` >= `quantity`. If not, return an error (e.g., HTTP 409 Conflict or specific business logic error indicating insufficient stock).
    *   If sufficient stock exists, the API must increment the `reserved_count` for the product and create a separate record in DynamoDB (or update the item with a list of reservations) with the `reservationId`, `quantity`, and a `reservation_expiry` TTL attribute (e.g., 30-60 minutes).
    *   The `DELETE /inventory/{productId}/reserve/{reservationId}` API endpoint must exist to release a specific reservation. This decrements `reserved_count` and removes the reservation record/entry.
    *   Upon successful checkout confirmation, a backend process (triggered by the order service) must permanently decrement `stock_count` by the purchased quantity and remove the associated reservation(s). This operation must also use conditional expressions (`stock_count` >= purchased quantity) to prevent over-selling in race conditions.
    *   DynamoDB TTL must automatically remove expired reservation records, and a mechanism (potentially a Lambda triggered by DynamoDB Streams or a periodic scan) must decrement `reserved_count` for expired reservations.
*   **Edge Cases:**
    *   Insufficient stock during reservation attempt: API returns specific error code/message.
    *   Reservation attempt for non-existent product: API returns 404 Not Found.
    *   Attempting to release a non-existent reservation: API returns 404 Not Found.
    *   Multiple users reserving the last item concurrently: DynamoDB write operations with conditional expressions will ensure only one succeeds in reserving the item, returning an error for others.
    *   User abandons cart: TTL handles cleanup.
    *   Checkout fails after reservation: Reservation remains until TTL expires or potentially released by a separate cleanup process depending on order state.
    *   Network issues during add-to-cart: Frontend should handle retries or inform the user.

### Feature 4.4: Low Stock Notifications

*   **User Stories:**
    *   As Mark, I want to receive an alert when the stock level for a product drops below its configured threshold so I can reorder in time.
*   **Acceptance Criteria:**
    *   A Lambda function (`checkThresholds`) must periodically scan the DynamoDB table.
    *   The scan should identify products where `stock_count` is less than or equal to `threshold_level` and `status` is not already "LOW_STOCK" or "OUT_OF_STOCK". (Alternatively, filter during the scan if possible, or use a GSI if scanning by status/stock range becomes too expensive - the current plan suggests a scan).
    *   For each identified product meeting the low stock criteria (and not previously alerted), the Lambda must:
        *   Update the item's `status` to "LOW_STOCK".
        *   Publish a message to the `inventory-alerts` SNS topic containing relevant product information (`product_id`, `warehouse_id`, current `stock_count`, `threshold_level`).
    *   The Lambda must be triggered automatically every 15 minutes by a CloudWatch Event Rule.
    *   An email subscription must be configured for the SNS topic to notify relevant personnel (e.g., `inventory-team@example.com`).
*   **Edge Cases:**
    *   A product's stock goes from above threshold directly to 0: It should trigger the low stock alert (as 0 <= threshold) and status update.
    *   Network issues prevent SNS publication: CloudWatch Logs should capture errors, and potentially retries should be implemented in the Lambda.
    *   DynamoDB scan performance: Monitor scan costs and consider alternative approaches (like DynamoDB Streams processing updates) if the table grows very large and scans become too expensive or slow. The GSI on `category` might help if scanning a subset is useful, but a full scan based on `stock_count <= threshold` likely requires iterating all items or a complex filter expression on a scan. A better approach for *very* large tables might be triggering checks on `PUT` via streams or checking on read if `status` is still "IN_STOCK" but calculated stock is low, then updating status and triggering SNS. *Stick to the plan (Scan) for V1 based on the prompt, but note this as a potential future enhancement.*

## 5. Technical Requirements

*   **API Endpoints:**
    *   `GET /inventory/{productId}`: Retrieve current stock details for a product.
        *   Parameters: `productId` (path)
        *   Response: JSON containing `product_id`, `warehouse_id`, `stock_count`, `reserved_count`, `threshold_level`, `status`, `last_updated`. HTTP 200 on success, 404 if not found, 500 on error.
    *   `PUT /inventory/{productId}`: Update stock details for a product.
        *   Parameters: `productId` (path)
        *   Request Body: JSON containing fields to update (e.g., `stock_count`, `threshold_level`), and a concurrency token (e.g., `expected_last_updated` timestamp or version number) for conditional update.
        *   Response: JSON of the updated record on HTTP 200. HTTP 400 on invalid input, 401/403 on auth failure, 404 if not found, 409 on concurrency conflict, 500 on error.
    *   `POST /inventory/{productId}/reserve`: Reserve stock for a product.
        *   Parameters: `productId` (path)
        *   Request Body: JSON containing `quantity` (number), `reservationId` (string, unique identifier for this specific reservation instance).
        *   Response: JSON confirming reservation details (e.g., `reservationId`, `quantity`, `expiry`) on HTTP 201. HTTP 400 on invalid input, 404 if product not found, 409 on insufficient stock, 500 on error.
    *   `DELETE /inventory/{productId}/reserve/{reservationId}`: Release a specific stock reservation.
        *   Parameters: `productId` (path), `reservationId` (path)
        *   Response: HTTP 204 on success. HTTP 404 if reservation not found for product, 500 on error.
*   **Data Storage:**
    *   **Database:** Amazon DynamoDB
    *   **Table Name:** `inventory-management`
    *   **Primary Key:** `product_id` (Partition Key, String), `warehouse_id` (Sort Key, String)
    *   **Global Secondary Index (GSI):**
        *   `category-index`: Partition Key `category` (String). Used for querying inventory by category (though not explicitly required by the listed features, it's in the plan).
    *   **Attributes:**
        *   `product_id` (String)
        *   `warehouse_id` (String)
        *   `stock_count` (Number): Current available stock.
        *   `reserved_count` (Number): Stock temporarily reserved (e.g., in active carts).
        *   `threshold_level` (Number): Stock level below which a low-stock alert is triggered.
        *   `last_updated` (String - ISO8601): Timestamp of the last inventory update. Used for conditional updates.
        *   `status` (String - ENUM): "IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK".
        *   `category` (String): Product category (for GSI).
        *   `reservation_expiry` (Number - Unix Timestamp): TTL attribute for reservation entries/records.
    *   **Consistency:** Reads for displaying stock (`GET`) must use Strong Consistency. Writes (`PUT`, `POST`, `DELETE`) are eventually consistent *within* the write operation, but conditional updates handle write conflicts.
    *   **Billing:** PAY_PER_REQUEST.
    *   **Reliability:** Point-in-Time Recovery enabled.
*   **AWS Services:**
    *   AWS Lambda: Execution environment for backend logic.
    *   Amazon API Gateway: Entry point for REST APIs, handles routing, authentication (API Keys), CORS.
    *   Amazon DynamoDB: NoSQL database for storing inventory data.
    *   Amazon S3: Hosting for static frontend files.
    *   Amazon SNS: Pub/Sub for low stock notifications.
    *   CloudWatch: Monitoring, logging, and scheduled event rules (for threshold check).
    *   IAM: Access control for AWS resources.
    *   Terraform: Infrastructure as Code for provisioning and managing AWS resources.
*   **Frontend Requirements:**
    *   A reusable JavaScript component (`inventory-widget.js`) to display stock status on product pages.
    *   An admin HTML page/component (`admin-panel.html`) to view and update inventory (requires secure authentication/authorization mechanisms beyond basic API keys for production).
    *   A JavaScript client library (`api-client.js`) to interact with the Inventory API Gateway endpoints.
    *   Bundling using Webpack.
    *   S3 bucket configured for static website hosting with appropriate CORS headers.
*   **Security:**
    *   API Gateway secured with API Keys for initial implementation (note: production admin interface needs stronger auth like Cognito).
    *   IAM roles with least privilege for Lambda functions.
    *   S3 bucket policy restricts access appropriately (public read for website, private for configurations/code).
*   **Monitoring & Logging:**
    *   CloudWatch Logs for Lambda execution logs.
    *   CloudWatch Alarms for API Gateway errors (e.g., 5xx errors).
    *   Monitoring DynamoDB performance (latency, throttled requests).
*   **Infrastructure as Code:**
    *   All AWS resources defined and managed using Terraform.
    *   Terraform state managed securely in an S3 bucket with versioning.

## 6. Implementation Roadmap

The implementation will follow a phased approach based on the provided breakdown:

**Phase 1: Foundation & Database (Issues #1 & #2)**
*   Set up core AWS infrastructure (VPC, Subnets, Security Groups if needed - *Note: Serverless services abstract much of this, focus on IAM, S3 state, Logging*).
*   Configure Terraform state management. (Optional for now)
*   Define and apply core IAM roles and resource tagging.
*   Create the DynamoDB table `inventory-management` with the specified schema, keys, GSI, TTL, and billing mode.
*   Enable Point-in-Time Recovery for DynamoDB.
*   **Target Completion:** Week 1

**Phase 2: Core Backend API (Issue #3)**
*   Develop and deploy the four core Lambda functions (`getInventoryLevel`, `updateInventoryLevel`, `reserveInventory`, `releaseInventory`).
*   Configure IAM execution roles for Lambdas with necessary DynamoDB permissions.
*   Set up the API Gateway REST API with resources, methods, and integrations pointing to Lambdas.
*   Implement API Key security and usage plans.
*   Configure CORS for API Gateway.
*   **Target Completion:** Week 2

**Phase 3: Notifications & Thresholds (Issue #4)**
*   Create the `inventory-alerts` SNS topic.
*   Develop and deploy the `checkThresholds` Lambda function.
*   Configure a CloudWatch Event Rule to trigger `checkThresholds` periodically (15 min).
*   Set up an initial email subscription for the SNS topic.
*   Test the notification flow.
*   **Target Completion:** Week 3

**Phase 4: Frontend Integration (Issue #5)**
*   Create the S3 bucket for static website hosting and configure the bucket policy.
*   Develop the `inventory-widget.js` component to call the `GET` API and display stock status on a sample product page.
*   Develop a basic `admin-panel.html` and `api-client.js` to interact with the `PUT` endpoint (using API Key).
*   Set up Webpack for bundling.
*   Deploy static files to S3.
*   **Target Completion:** Week 4

**Phase 5: Testing, Documentation & Deployment Scripts (Issues #6 & #7)**
*   Develop comprehensive API test cases (Postman/curl) covering success, failure, and edge cases (concurrency, insufficient stock, invalid input).
*   Write Lambda unit tests using test events.
*   Set up CloudWatch alarms for API Gateway errors.
*   Create initial documentation (README) covering architecture, API endpoints, setup, and DB schema.
*   Develop `init-db.js` script for populating test data.
*   Create a basic `deploy.sh` script to orchestrate Terraform apply steps.
*   Define Terraform outputs for key endpoints and resource names.
*   Conduct end-to-end testing (Frontend -> API -> DB -> Notifications).
*   **Target Completion:** Week 5

This roadmap allows for building the core backend logic and data model first, then adding notifications, integrating the frontend, and finally focusing on hardening, testing, and documentation before a potential release.