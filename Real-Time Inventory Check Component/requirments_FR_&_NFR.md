
# Real-Time Inventory Check Component for E-commerce

**Version:** 1.0
**Date:** May 6, 2025

## 1. Project Overview

This document outlines the requirements for a Real-Time Inventory Check Component, a key module designed for integration into an e-commerce platform. The primary objective is to eliminate "phantom inventory" issues by providing accurate, up-to-the-minute stock information to customers and administrators.

**Purpose:**
To ensure customers only see products as available if they are genuinely in stock, thereby improving customer satisfaction, reducing order cancellations, and enhancing operational efficiency for the e-commerce business.

**Goals:**
*   Provide customers with accurate, real-time stock visibility on product pages.
*   Enable temporary stock allocation during the customer checkout process.
*   Ensure accurate decrementing of stock upon successful order completion.
*   Notify administrators proactively when stock levels drop below predefined thresholds.
*   Provide administrators with secure tools/APIs to manage inventory levels.
*   Utilize a scalable, cost-effective serverless architecture on AWS.

**Target Users:**
*   E-commerce Customers (viewing stock)
*   E-commerce Administrators (managing stock, receiving notifications)
*   E-commerce Platform (integrating the component via APIs)

**Scope:**
This component focuses specifically on the inventory management aspects: checking, updating, reserving, releasing stock, and triggering low-stock notifications. It provides APIs for integration and a basic admin interface example. It does *not* include the full e-commerce checkout process, payment processing, shipping logic, or comprehensive warehouse management features, which are assumed to be handled by other systems integrating with this component.

**High level Architecture components:**
The component leverages a serverless AWS architecture, utilizing API Gateway as the entry point for REST APIs, Lambda functions for business logic, DynamoDB as the inventory database, SNS for notifications, S3 for static frontend assets, and CloudWatch for basic monitoring.

## 2. Functional Requirements

This section details the required features and functionalities of the Inventory Check Component.

**FR-1: Real-Time Stock Visibility**
*   **Description:** Customers viewing a product page shall see the current available stock level.
*   **Acceptance Criteria:**
    *   The system must query the current stock (`stock_count` - `reserved_count`) for the specific product upon page load.
    *   The displayed stock level must reflect the data stored in DynamoDB with strong consistency.
    *   The stock status ("IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK") based on `stock_count` and `threshold_level` must be displayed.
    *   Latency for retrieving stock information must be minimal (target < 200ms).
*   **Technical Mapping:** `getInventoryLevel` Lambda function via GET /inventory/{productId} API.

**FR-2: Inventory Update by Administrator**
*   **Description:** Administrators shall be able to update the total stock count for a product via a secure interface or API.
*   **Acceptance Criteria:**
    *   An administrator can specify a new total `stock_count` for a given `product_id` and `warehouse_id`.
    *   The system must use a conditional expression (e.g., checking a version or ETag) to prevent concurrent update conflicts.
    *   The `last_updated` timestamp must be recorded.
    *   The product `status` must be updated based on the new `stock_count` and `threshold_level`.
    *   The update must be performed via a secured endpoint requiring authentication (e.g., API Key).
*   **Technical Mapping:** `updateInventoryLevel` Lambda function via PUT /inventory/{productId} API.

**FR-3: Temporary Inventory Reservation**
*   **Description:** When a customer adds an item to their cart or proceeds to checkout, the system must temporarily reserve the requested quantity.
*   **Acceptance Criteria:**
    *   The system must decrement the `stock_count` and increment the `reserved_count` for the specified product and quantity.
    *   The system must ensure that the requested quantity does not exceed the available stock (`stock_count` - `reserved_count`) *before* the reservation. If it does, the request must fail.
    *   A Time-to-Live (TTL) must be set on the reservation record/item to automatically release the reservation after a configured period (e.g., 15-30 minutes).
    *   The API must return a reservation identifier (`reservationId`).
*   **Technical Mapping:** `reserveInventory` Lambda function via POST /inventory/{productId}/reserve API, using TTL attribute `reservation_expiry`.

**FR-4: Release Reserved Inventory**
*   **Description:** Reserved inventory must be released back to available stock if a reservation expires or is explicitly canceled (e.g., item removed from cart, order canceled).
*   **Acceptance Criteria:**
    *   If a reservation expires via TTL, the system must automatically return the reserved quantity (`reserved_count`) for that specific reservation back into the available stock (`stock_count`). *Note: The breakdown implies TTL on the item itself, this needs careful design to only release *that specific reservation's* quantity if multiple reservations exist for one product.* A common pattern is a separate 'reservations' table. Assuming the provided breakdown's single item approach, this criteria needs refinement, perhaps focusing on manual release via API. *Revision based on breakdown:* Focus on the API release.
    *   An external system (e.g., e-commerce platform) must be able to explicitly call an API to release a specific reserved quantity, identified by a `reservationId`.
    *   The system must decrement the `reserved_count` and increment the `stock_count` for the specified product and reservation ID.
*   **Technical Mapping:** `releaseInventory` Lambda function via DELETE /inventory/{productId}/reserve/{reservationId} API. TTL handling for expiry needs further design beyond the simple schema provided.

**FR-5: Permanent Inventory Decrement (Checkout Confirmation)**
*   **Description:** Upon successful checkout confirmation by the e-commerce platform, the system must permanently decrement the inventory for the purchased items.
*   **Acceptance Criteria:**
    *   An external system (e.g., order processing) must call an API specifying the product(s) and quantity purchased.
    *   The system must decrement the `stock_count` for each purchased item.
    *   If items were previously reserved (FR-3), the corresponding `reserved_count` must also be decremented as these items are no longer merely reserved but sold.
    *   The update must ensure sufficient *reserved* quantity exists for the purchased items if decrementing from reserved stock.
    *   The product `status` must be updated based on the new `stock_count`.
*   **Technical Mapping:** This maps closest to a specific use case of the `updateInventoryLevel` or potentially a dedicated 'fulfillOrder' Lambda/API endpoint (not explicitly in breakdown but needed). Let's assume `updateInventoryLevel` is used with appropriate parameters or a new Lambda is added. *Revision:* Given the breakdown, this action likely uses `updateInventoryLevel` to decrement `stock_count` and `releaseInventory` (or a variation) to clear the `reserved_count`. Let's define it as a distinct *functional need* regardless of the final API endpoint name.
    *   Acceptance Criteria (Refined):
        *   An external system calls an API (e.g., `fulfillInventory`) with order details (`product_id`, `warehouse_id`, quantity, potentially `reservationId`).
        *   The system atomically decrements the `stock_count` by the purchased quantity.
        *   If a `reservationId` is provided, the corresponding `reserved_count` is also decremented.
        *   The product `status` is updated.

**FR-6: Low Stock Notifications**
*   **Description:** Administrators must be notified when a product's available stock (`stock_count`) falls below its configured `threshold_level`.
*   **Acceptance Criteria:**
    *   A scheduled process must periodically check stock levels for all products.
    *   If `stock_count` <= `threshold_level`, a notification must be published.
    *   The notification must contain details including product ID, warehouse ID, current stock, and threshold.
    *   The product `status` must be updated to "LOW_STOCK".
    *   Notifications must be sent via SNS (e.g., emailed to subscribed administrators).
    *   Notifications should ideally trigger only once per transition from "IN_STOCK" or "OUT_OF_STOCK" to "LOW_STOCK", not on every check if it remains low. *Refinement needed for this criteria based on breakdown's simple scan/publish.* Let's stick to the breakdown:
    *   Acceptance Criteria (Refined based on breakdown):
        *   A process runs periodically (e.g., every 15 minutes).
        *   This process identifies products where `stock_count` <= `threshold_level`.
        *   For identified products, a message is published to an SNS topic.
        *   The product `status` is updated to "LOW_STOCK".
        *   Subscribers to the SNS topic receive the notification.
*   **Technical Mapping:** `checkThresholds` Lambda function triggered by CloudWatch Event Rule, publishing to `inventory-alerts` SNS topic.

**FR-7: Administrator Interface (Basic)**
*   **Description:** Provide a basic web interface for administrators to view and update inventory levels.
*   **Acceptance Criteria:**
    *   The interface must load from the S3 frontend bucket.
    *   Administrators can search/view inventory levels for products.
    *   Administrators can input and submit new stock quantities using the FR-2 functionality.
    *   The interface must utilize the defined Inventory API endpoints (`api-client.js`).
*   **Technical Mapping:** Static files hosted on S3 (`admin-panel.html`, `inventory-widget.js`, `api-client.js`).

## 3. Non-Functional Requirements

This section details the quality attributes and technical constraints of the Inventory Check Component.

**NFR-1: Performance**
*   **Response Time:** API responses for viewing and reserving stock should be fast (target < 300ms under normal load).
*   **Throughput:** The system must handle concurrent inventory check and reservation requests typical for an e-commerce platform during peak times. Serverless architecture is expected to scale automatically.
*   **Batch Processing:** The threshold check Lambda should complete its scan and notification process efficiently within its execution time limit.

**NFR-2: Security**
*   **API Access:** API endpoints must be secured. The breakdown suggests API Keys, which is a form of authentication, though more robust mechanisms like OAuth or Cognito might be considered for production. For this component, API Key requirement from breakdown is sufficient.
*   **Least Privilege:** AWS Lambda execution roles must adhere to the principle of least privilege, granting access only to necessary DynamoDB operations, SNS publishing, and CloudWatch logs.
*   **Data Protection:** Inventory data in DynamoDB must be encrypted at rest (AWS default encryption is acceptable).
*   **CORS:** API Gateway must be configured with appropriate CORS headers to allow access from the frontend domain.
*   **Admin Interface:** Access to the admin interface (`admin-panel.html`) or the underlying APIs it calls for updates should be restricted (e.g., via API Key or future integration with an authentication system).

**NFR-3: Reliability and Availability**
*   **Uptime:** The serverless components (Lambda, API Gateway, DynamoDB, SNS) are managed services designed for high availability. The system relies on AWS's inherent redundancy.
*   **Data Durability:** DynamoDB provides built-in data durability. Point-in-Time Recovery (PITR) should be enabled as per the breakdown to allow restoring data to a previous state.
*   **Monitoring:** Basic monitoring using CloudWatch logs and metrics should be enabled to track API calls, Lambda errors, and DynamoDB activity. Alarms for critical errors (e.g., API Gateway 5xx errors) must be configured.

**NFR-4: Scalability**
*   **Elasticity:** The serverless architecture is inherently elastic, scaling automatically based on demand for API calls and Lambda executions.
*   **Database:** DynamoDB's PAY_PER_REQUEST billing mode ensures the database scales throughput automatically based on traffic volume without capacity planning.

**NFR-5: Maintainability**
*   **Infrastructure as Code (IaC):** The entire AWS infrastructure must be defined using Terraform for repeatability and version control.
*   **Modularity:** Terraform code and Lambda functions should be structured in a modular way as suggested by the breakdown.
*   **Logging:** Lambda functions must log relevant information to CloudWatch Logs for debugging and monitoring.

**NFR-6: Usability**
*   **API Documentation:** API endpoints, request/response formats, and error codes must be clearly documented (e.g., in README.md).
*   **Admin Interface:** The basic admin interface should be intuitive for simple viewing and updating tasks.

**NFR-7: Technical Requirements**
*   **AWS Services:** Must be deployable on AWS using the specified services: Lambda, API Gateway, DynamoDB, S3, SNS, CloudWatch.
*   **Database Schema:** Must adhere to the defined DynamoDB schema (product_id, warehouse_id, stock_count, reserved_count, etc.) with necessary indexes (GSI1: category-index).
*   **Consistency:** Inventory checks triggered by customer views (FR-1) must use strong consistency reads from DynamoDB to ensure the absolute latest stock count is displayed.
*   **Lambda Runtimes:** Lambda functions will be implemented using Python or Node.js as per the breakdown.

## 4. Dependencies and Constraints

**Dependencies:**
*   **E-commerce Platform:** The component requires integration with an existing e-commerce platform that will:
    *   Call the GET API (FR-1) on product pages.
    *   Call the POST API (FR-3) for reserving stock.
    *   Call the DELETE API (FR-4) for releasing stock (e.g., cart removal).
    *   Call the "fulfill" API (FR-5) upon order confirmation.
    *   Integrate the S3-hosted frontend components (FR-7).
    *   Handle all other e-commerce logic (user accounts, checkout flow, payment, shipping).
*   **AWS Cloud:** The component is entirely dependent on the availability and functionality of the specified AWS services.
*   **Internet Connectivity:** Required for users to access the frontend and for the e-commerce platform to interact with the APIs.

**Constraints:**
*   **Scope Limitation:** The component is strictly focused on inventory management logic as defined. It does not encompass full warehouse management or other broader supply chain processes.
*   **Basic Admin UI:** The provided S3 frontend is a *basic* example (FR-7) and may require further development by the integrating platform for a production-ready admin experience.
*   **Reservation Model:** The provided schema implies reservations are tracked potentially within the main product item (via `reserved_count`). This model can become complex with multiple reservations for the same product. A separate reservation entity/table might be more robust for complex scenarios, but the requirements align with the provided single-table, reserved_count approach for this version.
*   **Notification Granularity:** The threshold check triggers based on the *total* stock vs. threshold. More granular notifications (e.g., per warehouse) would require schema or logic modifications beyond the current breakdown.

## 5. Risk Assessment

**Risk 1: Data Inconsistency due to Race Conditions**
*   **Description:** High volume of concurrent requests (especially reserve/update) could potentially lead to slightly inaccurate stock counts despite using strong consistency and conditional updates, if the application logic isn't perfectly robust against all possible interleavings. The simple `reserved_count` on the item might not fully protect against issues if multiple users reserve simultaneously without a reservation ID lock mechanism.
*   **Likelihood:** Medium
*   **Impact:** High (Leads to phantom inventory or overselling).
*   **Mitigation:** Thorough unit and integration testing of Lambda logic, particularly for `reserveInventory` and `updateInventoryLevel` with concurrent access. Consider adding a dedicated 'reservations' table with transaction support if the single-item model proves insufficient under stress testing. (TODO - to check possibilities of DynamoDB transactions or conditional writes to ensure atomicity).

**Risk 2: API Performance Degradation**
*   **Description:** While serverless scales, sudden, massive spikes in traffic *could* potentially strain DynamoDB (though PAY_PER_REQUEST helps), or the Lambda cold starts *could* introduce latency under certain traffic patterns.
*   **Likelihood:** Low (AWS handles most scaling) to Medium (if traffic patterns are highly unpredictable).
*   **Impact:** Medium (Slow customer experience, potential timeouts).
*   **Mitigation:** Implement detailed CloudWatch monitoring for Lambda duration, API Gateway latency, and DynamoDB metrics. Utilize provisioned concurrency for critical Lambda functions if cold starts become an issue. Optimize DynamoDB queries (ensure efficient use of partition/sort keys and GSI).

**Risk 3: Security Breach (API Key Compromise)**
*   **Description:** If the API Key for the update/reserve endpoints is compromised, malicious actors could potentially manipulate inventory levels.
*   **Likelihood:** Medium (Depends on how keys are managed by the integrating platform).
*   **Impact:** High (Data integrity loss, potential financial impact).
*   **Mitigation:** Strongly recommend the integrating platform securely manages and stores API Keys (e.g., using AWS Secrets Manager). Explore upgrading security mechanism to OAuth/Cognito in future versions for better user/client identification and authorization. Enforce least privilege on IAM roles.

**Risk 4: Cost Overruns**
*   **Description:** Unexpectedly high traffic could lead to significant costs for Lambda executions, API Gateway requests, and DynamoDB usage under PAY_PER_REQUEST.
*   **Likelihood:** Low to Medium (Depends on business success and traffic prediction).
*   **Impact:** Medium to High (Unexpected operational expense).
*   **Mitigation:** Set up CloudWatch billing alarms. Regularly review AWS usage reports. Monitor Lambda durations and DynamoDB consumed capacity. Optimize Lambda code for efficiency.

**Risk 5: Integration Challenges with E-commerce Platform**
*   **Description:** Integrating the APIs and frontend components into an existing, potentially complex, e-commerce platform might face technical hurdles (e.g., incompatible frameworks, data mapping issues, coordinating API calls within their checkout flow).
*   **Likelihood:** Medium to High (Integration is rarely trivial).
*   **Impact:** Medium (Delays project timeline, requires refactoring).
*   **Mitigation:** Provide clear API documentation (FR-6). Develop comprehensive integration guides. Offer technical support during the integration phase. Design APIs to be as flexible and decoupled as possible.

**Risk 6: Notification Failures**
*   **Description:** The periodic check Lambda could fail, or SNS could experience issues, leading to administrators not receiving low stock alerts.
*   **Likelihood:** Low (AWS services are reliable) to Medium (Lambda execution errors are possible).
*   **Impact:** Medium (Administrators are unaware of low stock, potentially leading to stockouts).
*   **Mitigation:** Implement CloudWatch alarms for the `checkThresholds` Lambda errors and invocations. Monitor SNS delivery status. Ensure Lambda IAM role has correct SNS publish permissions.

**Risk 7: TTL Configuration or Logic Error**
*   **Description:** Misconfiguring the TTL attribute or relying solely on TTL without a robust reservation ID system could lead to reserved items not being released correctly or being released prematurely/late.
*   **Likelihood:** Medium (TTL requires careful setup and monitoring).
*   **Impact:** Medium (Inaccurate available stock counts).
*   **Mitigation:** Thorough testing of reservation and release flows under various scenarios, including waiting for TTL expiry. Document the TTL configuration and monitoring process. The provided schema's simple `reserved_count` might require a more complex reservation item structure in DynamoDB for true TTL-based expiry per reservation.


