# Requirements Function And Non-Functional Document: Real-Time Inventory Check Component

## Project Overview

This document outlines the requirements for a Real-Time Inventory Check Component, a key module designed for integration into an e-commerce platform. The primary objective is to eliminate "phantom inventory" issues by providing accurate, up-to-the-minute stock information to customers and administrators.

### Purpose

To ensure customers only see products as available if they are genuinely in stock, thereby improving customer satisfaction, reducing order cancellations, and enhancing operational efficiency for the e-commerce business.

### Goals

- Provide customers with accurate, real-time stock visibility on product pages
- Enable temporary stock allocation during the customer checkout process
- Ensure accurate decrementing of stock upon successful order completion
- Notify administrators proactively when stock levels drop below predefined thresholds
- Provide administrators with secure tools/APIs to manage inventory levels
- Utilize a scalable, cost-effective serverless architecture on AWS

### Target Users

- E-commerce Customers (viewing stock)
- E-commerce Administrators (managing stock, receiving notifications)
- E-commerce Platform (integrating the component via APIs)

### Scope

This component focuses specifically on the inventory management aspects: checking, updating, reserving, releasing stock, and triggering low-stock notifications. It provides APIs for integration and a basic admin interface example. It does not include the full e-commerce checkout process, payment processing, shipping logic, or comprehensive warehouse management features.

## Functional Requirements

### FR-1: Real-Time Stock Visibility

**Description:** Customers viewing a product page shall see the current available stock level.

**Acceptance Criteria:**
- The system must query the current stock (stock_count - reserved_count) for the specific product upon page load
- The displayed stock level must reflect the data stored in DynamoDB with strong consistency
- The stock status ("IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK") based on stock_count and threshold_level must be displayed
- Latency for retrieving stock information must be minimal (target < 200ms)

**Technical Mapping:** `getInventoryLevel` Lambda function via GET `/inventory/{productId}` API

### FR-2: Inventory Update by Administrator

**Description:** Administrators shall be able to update the total stock count for a product via a secure interface or API.

**Acceptance Criteria:**
- An administrator can specify a new total stock_count for a given product_id and warehouse_id
- The system must use a conditional expression to prevent concurrent update conflicts
- The last_updated timestamp must be recorded
- The product status must be updated based on the new stock_count and threshold_level
- The update must be performed via a secured endpoint requiring authentication

**Technical Mapping:** `updateInventoryLevel` Lambda function via PUT `/inventory/{productId}` API

### FR-3: Temporary Inventory Reservation

**Description:** When a customer adds an item to their cart or proceeds to checkout, the system must temporarily reserve the requested quantity.

**Acceptance Criteria:**
- The system must decrement the stock_count and increment the reserved_count for the specified product and quantity
- The system must ensure that the requested quantity does not exceed the available stock (stock_count - reserved_count)
- A Time-to-Live (TTL) must be set on the reservation record/item to automatically release the reservation after a configured period
- The API must return a reservation identifier (reservationId)

**Technical Mapping:** `reserveInventory` Lambda function via POST `/inventory/{productId}/reserve` API

### FR-4: Release Reserved Inventory

**Description:** Reserved inventory must be released back to available stock if a reservation expires or is explicitly canceled.

**Acceptance Criteria:**
- An external system must be able to explicitly call an API to release a specific reserved quantity, identified by a reservationId
- The system must decrement the reserved_count and increment the stock_count for the specified product and reservation ID

**Technical Mapping:** `releaseInventory` Lambda function via DELETE `/inventory/{productId}/reserve/{reservationId}` API

### FR-5: Permanent Inventory Decrement (Checkout Confirmation)

**Description:** Upon successful checkout confirmation by the e-commerce platform, the system must permanently decrement the inventory for the purchased items.

**Acceptance Criteria:**
- An external system must call an API specifying the product(s) and quantity purchased
- The system must decrement the stock_count for each purchased item
- If items were previously reserved, the corresponding reserved_count must also be decremented
- The update must ensure sufficient reserved quantity exists for the purchased items if decrementing from reserved stock
- The product status must be updated based on the new stock_count

**Technical Mapping:** Using `updateInventoryLevel` with appropriate parameters

### FR-6: Low Stock Notifications

**Description:** Administrators must be notified when a product's available stock falls below its configured threshold_level.

**Acceptance Criteria:**
- A scheduled process must periodically check stock levels for all products
- If stock_count <= threshold_level, a notification must be published
- The notification must contain details including product ID, warehouse ID, current stock, and threshold
- The product status must be updated to "LOW_STOCK"
- Notifications must be sent via SNS

**Technical Mapping:** `checkThresholds` Lambda function triggered by CloudWatch Event Rule


### FR-7: Administrator Interface (Basic)

**Description:** Provide a basic web interface for administrators to view and update inventory levels.

**Acceptance Criteria:**
- The interface must load from the S3 frontend bucket
- Administrators can search/view inventory levels for products
- Administrators can input and submit new stock quantities
- The interface must utilize the defined Inventory API endpoints

**Technical Mapping:** Static files hosted on S3

## Non-Functional Requirements

### NFR-1: Performance

- **Response Time:** API responses for viewing and reserving stock should be fast (target < 300ms under normal load)
- **Throughput:** The system must handle concurrent inventory check and reservation requests typical for an e-commerce platform during peak times
- **Batch Processing:** The threshold check Lambda should complete its scan and notification process efficiently within its execution time limit

### NFR-2: Security

- **API Access:** API endpoints must be secured with API Keys
- **Least Privilege:** AWS Lambda execution roles must adhere to the principle of least privilege
- **Data Protection:** Inventory data in DynamoDB must be encrypted at rest
- **CORS:** API Gateway must be configured with appropriate CORS headers
- **Admin Interface:** Access to the admin interface or the underlying APIs it calls for updates should be restricted

### NFR-3: Reliability and Availability

- **Uptime:** The serverless components are managed services designed for high availability
- **Data Durability:** DynamoDB provides built-in data durability. Point-in-Time Recovery should be enabled
- **Monitoring:** Basic monitoring using CloudWatch logs and metrics should be enabled

### NFR-4: Scalability

- **Elasticity:** The serverless architecture is inherently elastic, scaling automatically based on demand
- **Database:** DynamoDB's PAY_PER_REQUEST billing mode ensures the database scales throughput automatically

### NFR-5: Maintainability

- **Infrastructure as Code:** The entire AWS infrastructure must be defined using Terraform
- **Modularity:** Terraform code and Lambda functions should be structured in a modular way
- **Logging:** Lambda functions must log relevant information to CloudWatch Logs


### NFR-6: Usability

- **API Documentation:** API endpoints, request/response formats, and error codes must be clearly documented
- **Admin Interface:** The basic admin interface should be intuitive for simple viewing and updating tasks

### NFR-7: Technical Requirements

- **AWS Services:** Must be deployable on AWS using Lambda, API Gateway, DynamoDB, S3, SNS, CloudWatch
- **Database Schema:** Must adhere to the defined DynamoDB schema with necessary indexes
- **Consistency:** Inventory checks must use strong consistency reads from DynamoDB
- **Lambda Runtimes:** Lambda functions will be implemented using Python or Node.js

## Technical Requirements

### API Endpoints

- GET `/inventory/{productId}`: Retrieve current stock details for a product
- PUT `/inventory/{productId}`: Update stock details for a product
- POST `/inventory/{productId}/reserve`: Reserve stock for a product
- DELETE `/inventory/{productId}/reserve/{reservationId}`: Release a specific stock reservation

### Data Storage

**Database:** Amazon DynamoDB
- **Table Name:** `inventory-management`
- **Primary Key:**
    - Partition Key: `product_id`
    - Sort Key: `warehouse_id`
- **Global Secondary Index:** `category-index`
- **Key Attributes:**
    - `product_id`
    - `warehouse_id`
    - `stock_count`
    - `reserved_count`
    - `threshold_level`
    - `last_updated`
    - `status`
    - `category`
    - `reservation_expiry`
- **Configuration:**
    - Consistency: Strong Consistency for reads
    - Billing: PAY_PER_REQUEST
    - Reliability: Point-in-Time Recovery enabled

### AWS Services

- AWS Lambda
- Amazon API Gateway
- Amazon DynamoDB
- Amazon S3
- Amazon SNS
- CloudWatch
- IAM
- Terraform

### Frontend Requirements

- JavaScript component (`inventory-widget.js`)
- Admin HTML page (`admin-panel.html`)
- JavaScript client library (`api-client.js`)

### Security

- API Gateway secured with API Keys
- IAM roles with least privilege
- S3 bucket policy with appropriate restrictions

### Monitoring & Logging

- CloudWatch Logs for Lambda execution logs
- CloudWatch Alarms for API Gateway errors
- DynamoDB performance monitoring

### Infrastructure as Code

- All resources defined and managed using Terraform
- Terraform state managed securely

## User Stories

### Customer-Facing Features

- **US-1:** As a customer, I want to see accurate stock levels on product pages so I can make informed purchase decisions
- **US-2:** As a customer, I want my selected items to be reserved during checkout so they don't sell out while I'm completing my purchase
- **US-3:** As a customer, I want to see different stock status indicators (In Stock, Low Stock, Out of Stock) so I can quickly understand product availability

### Administrator Features

- **US-4:** As an administrator, I want to update inventory levels for products so I can keep stock information accurate
- **US-5:** As an administrator, I want to receive notifications when stock drops below thresholds so I can reorder products
- **US-6:** As an administrator, I want to view current inventory levels across products so I can monitor stock status
- **US-7:** As an administrator, I want to set threshold levels for products so I can customize when low stock alerts are triggered

### E-commerce Platform Integration

- **US-8:** As a platform developer, I want clear API documentation so I can integrate the inventory component with our e-commerce platform
- **US-9:** As a platform developer, I want to call the reservation API during checkout so customer selections are temporarily secured
- **US-10:** As a platform developer, I want to call the release API when items are removed from cart so inventory is accurately reflected
- **US-11:** As a platform developer, I want to call the fulfill API when orders are confirmed so inventory is permanently decremented