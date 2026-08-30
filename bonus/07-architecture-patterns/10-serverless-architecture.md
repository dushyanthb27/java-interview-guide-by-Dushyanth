# Serverless Architecture

[Home](../../README.md) > Bonus > Architecture Patterns > Serverless Architecture

---

## Overview
Serverless architecture is a cloud-native execution model where the cloud provider dynamically manages the allocation of machine resources. Developers deploy code as functions or services, and the provider automatically scales, provisions, and manages the infrastructure. This model eliminates the need to provision and manage servers, allowing teams to focus on business logic.

## Pattern Description

In serverless architecture, applications are broken down into individual functions that execute in response to events. These functions are stateless, ephemeral, and scale automatically based on demand. The cloud provider handles all infrastructure management, including scaling, patching, and high availability.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Layer                             │
│  (Web, Mobile, IoT, Third-party APIs)                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / CDN                         │
│              (Routing, Auth, Caching, Rate Limiting)         │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Function   │ │  Function   │ │  Function   │ │  Function   │
│  (Auth)     │ │  (Process)  │ │  (Compute)  │ │  (Notify)  │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │               │
       ▼               ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Database   │ │  Database   │ │  Cache      │ │  Message    │
│  (DynamoDB) │ │  (RDS)      │ │  (Redis)    │ │  (SQS/SNS)  │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

        │              │              │              │
        └──────────────┼──────────────┼──────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Storage    │ │  Monitoring │ │  Logging    │
│  (S3)       │ │  (CloudWatch)│ │  (CloudTrail)│
└─────────────┘ └─────────────┘ └─────────────┘

Serverless Components:

1. Function as a Service (FaaS)
   ┌─────────────┐
   │   Function  │
   │   (Code)    │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  Runtime    │
   │  (Managed)  │
   └─────────────┘

2. Backend as a Service (BaaS)
   ┌─────────────┐
   │  Database   │
   │  (Managed)  │
   └─────────────┘

3. Event Sources
   ┌─────────────┐
   │  HTTP       │
   │  Storage    │
   │  Queue      │
   │  Schedule   │
   └─────────────┘
```

### Key Concepts

**Function**
- Single-purpose code unit
- Stateless execution
- Event-driven invocation
- Auto-scaling
- Pay-per-use pricing

**Event Source**
- Triggers function execution
- HTTP requests, file uploads, database changes
- Scheduled events, message queues
- Can be synchronous or asynchronous

**Cold Start**
- First invocation after idle period
- Takes longer to respond
- Can be mitigated with provisioned concurrency
- Important for latency-sensitive applications

**Statelessness**
- Functions don't maintain state
- State stored in external services
- Enables horizontal scaling
- Simplifies deployment

**Pay-per-Use**
- Pay only for execution time
- No cost when not running
- Predictable scaling costs
- Cost optimization important

## When to Use Serverless Architecture

### Ideal Scenarios
✅ Sporadic or unpredictable workloads
✅ Rapid prototyping and MVP development
✅ When operational overhead needs minimization
✅ Cost-sensitive applications with variable load
✅ Event-driven applications
✅ Microservices with independent scaling
✅ Teams with limited DevOps resources

### When to Avoid
❌ Long-running processes (timeout limits)
❌ High-performance computing requirements
❌ Predictable, continuous workloads (may be cheaper with servers)
❌ Strict regulatory requirements (data residency)
❌ Complex stateful applications
❌ When vendor lock-in is a concern

## Java Implementation

### Project Structure
```
serverless-app/
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── example/
│                   └── serverless/
│                       ├── api/
│                       │   ├── ApiGatewayHandler.java
│                       │   ├── ProductHandler.java
│                       │   └── OrderHandler.java
│                       ├── service/
│                       │   ├── ProductService.java
│                       │   └── OrderService.java
│                       ├── model/
│                       │   ├── Product.java
│                       │   ├── Order.java
│                       │   └── ApiRequest.java
│                       ├── repository/
│                       │   ├── DynamoDBProductRepository.java
│                       │   └── DynamoDBOrderRepository.java
│                       └── config/
│                           └── DynamoDBConfig.java
├── template.yaml  (AWS SAM template)
├── pom.xml
└── serverless.yml (Serverless Framework)
```

### API Gateway Handler
```java
package com.example.serverless.api;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.fasterxml.jackson.databind.ObjectMapper;

public class ApiGatewayHandler implements RequestHandler<ApiRequest, ApiResponse> {
    
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final ProductHandler productHandler = new ProductHandler();
    private final OrderHandler orderHandler = new OrderHandler();

    @Override
    public ApiResponse handleRequest(ApiRequest request, Context context) {
        try {
            String path = request.getPath();
            String method = request.getHttpMethod();

            switch (path) {
                case "/products":
                    if ("GET".equals(method)) {
                        return productHandler.getProducts(request, context);
                    } else if ("POST".equals(method)) {
                        return productHandler.createProduct(request, context);
                    }
                    break;
                case "/products/{id}":
                    if ("GET".equals(method)) {
                        return productHandler.getProduct(request, context);
                    }
                    break;
                case "/orders":
                    if ("POST".equals(method)) {
                        return orderHandler.createOrder(request, context);
                    }
                    break;
                default:
                    return new ApiResponse(404, "Not Found");
            }

            return new ApiResponse(405, "Method Not Allowed");

        } catch (Exception e) {
            context.getLogger().log("Error: " + e.getMessage());
            return new ApiResponse(500, "Internal Server Error");
        }
    }
}

package com.example.serverless.api;

public record ApiRequest(
    String path,
    String httpMethod,
    String body,
    String queryStringParameters,
    Map<String, String> headers,
    Map<String, String> pathParameters
) {
    public ApiRequest() {
        this(null, null, null, null, null, null);
    }
}

public record ApiResponse(
    int statusCode,
    String body,
    Map<String, String> headers
) {
    public ApiResponse(int statusCode, String body) {
        this(statusCode, body, Map.of("Content-Type", "application/json"));
    }
}
```

### Product Handler
```java
package com.example.serverless.api;

import com.amazonaws.services.lambda.runtime.Context;
import com.example.serverless.model.Product;
import com.example.serverless.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;

public class ProductHandler {
    
    private final ProductService productService = new ProductService();
    private final ObjectMapper objectMapper = new ObjectMapper();

    public ApiResponse getProducts(ApiRequest request, Context context) {
        try {
            List<Product> products = productService.getAllProducts();
            String jsonBody = objectMapper.writeValueAsString(products);
            return new ApiResponse(200, jsonBody);
        } catch (Exception e) {
            context.getLogger().log("Error getting products: " + e.getMessage());
            return new ApiResponse(500, "Error retrieving products");
        }
    }

    public ApiResponse getProduct(ApiRequest request, Context context) {
        try {
            String productId = request.pathParameters().get("id");
            Product product = productService.getProduct(productId);
            
            if (product == null) {
                return new ApiResponse(404, "Product not found");
            }

            String jsonBody = objectMapper.writeValueAsString(product);
            return new ApiResponse(200, jsonBody);
        } catch (Exception e) {
            context.getLogger().log("Error getting product: " + e.getMessage());
            return new ApiResponse(500, "Error retrieving product");
        }
    }

    public ApiResponse createProduct(ApiRequest request, Context context) {
        try {
            Product product = objectMapper.readValue(
                request.body(), 
                Product.class
            );
            
            Product created = productService.createProduct(product);
            String jsonBody = objectMapper.writeValueAsString(created);
            return new ApiResponse(201, jsonBody);
        } catch (Exception e) {
            context.getLogger().log("Error creating product: " + e.getMessage());
            return new ApiResponse(500, "Error creating product");
        }
    }
}
```

### Order Handler
```java
package com.example.serverless.api;

import com.amazonaws.services.lambda.runtime.Context;
import com.example.serverless.model.Order;
import com.example.serverless.service.OrderService;
import com.fasterxml.jackson.databind.ObjectMapper;

public class OrderHandler {
    
    private final OrderService orderService = new OrderService();
    private final ObjectMapper objectMapper = new ObjectMapper();

    public ApiResponse createOrder(ApiRequest request, Context context) {
        try {
            Order order = objectMapper.readValue(
                request.body(), 
                Order.class
            );
            
            Order created = orderService.createOrder(order);
            String jsonBody = objectMapper.writeValueAsString(created);
            return new ApiResponse(201, jsonBody);
        } catch (Exception e) {
            context.getLogger().log("Error creating order: " + e.getMessage());
            return new ApiResponse(500, "Error creating order");
        }
    }
}
```

### Service Layer
```java
package com.example.serverless.service;

import com.example.serverless.model.Product;
import com.example.serverless.repository.DynamoDBProductRepository;
import java.util.List;

public class ProductService {
    
    private final DynamoDBProductRepository productRepository = 
        new DynamoDBProductRepository();

    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    public Product getProduct(String id) {
        return productRepository.findById(id);
    }

    public Product createProduct(Product product) {
        return productRepository.save(product);
    }

    public Product updateProduct(String id, Product product) {
        return productRepository.update(id, product);
    }

    public void deleteProduct(String id) {
        productRepository.delete(id);
    }
}

package com.example.serverless.service;

import com.example.serverless.model.Order;
import com.example.serverless.repository.DynamoDBOrderRepository;

public class OrderService {
    
    private final DynamoDBOrderRepository orderRepository = 
        new DynamoDBOrderRepository();

    public Order createOrder(Order order) {
        // Business logic for order creation
        return orderRepository.save(order);
    }

    public Order getOrder(String id) {
        return orderRepository.findById(id);
    }
}
```

### DynamoDB Repository
```java
package com.example.serverless.repository;

import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBMapper;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBQueryExpression;
import com.example.serverless.model.Product;
import java.util.List;

public class DynamoDBProductRepository {
    
    private final DynamoDBMapper dynamoDBMapper;

    public DynamoDBProductRepository() {
        AmazonDynamoDB client = AmazonDynamoDBClientBuilder.defaultClient();
        this.dynamoDBMapper = new DynamoDBMapper(client);
    }

    public Product save(Product product) {
        dynamoDBMapper.save(product);
        return product;
    }

    public Product findById(String id) {
        return dynamoDBMapper.load(Product.class, id);
    }

    public List<Product> findAll() {
        return dynamoDBMapper.scan(Product.class, new DynamoDBScanExpression());
    }

    public Product update(String id, Product product) {
        product.setId(id);
        dynamoDBMapper.save(product);
        return product;
    }

    public void delete(String id) {
        Product product = findById(id);
        if (product != null) {
            dynamoDBMapper.delete(product);
        }
    }
}

package com.example.serverless.repository;

import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBMapper;
import com.example.serverless.model.Order;

public class DynamoDBOrderRepository {
    
    private final DynamoDBMapper dynamoDBMapper;

    public DynamoDBOrderRepository() {
        AmazonDynamoDB client = AmazonDynamoDBClientBuilder.defaultClient();
        this.dynamoDBMapper = new DynamoDBMapper(client);
    }

    public Order save(Order order) {
        dynamoDBMapper.save(order);
        return order;
    }

    public Order findById(String id) {
        return dynamoDBMapper.load(Order.class, id);
    }
}
```

### Domain Models
```java
package com.example.serverless.model;

import com.amazonaws.services.dynamodbv2.datamodeling.*;

@DynamoDBTable(tableName = "Products")
public class Product {
    @DynamoDBHashKey(attributeName = "id")
    private String id;
    
    @DynamoDBAttribute(attributeName = "name")
    private String name;
    
    @DynamoDBAttribute(attributeName = "description")
    private String description;
    
    @DynamoDBAttribute(attributeName = "price")
    private double price;
    
    @DynamoDBAttribute(attributeName = "stockQuantity")
    private int stockQuantity;

    // Constructors, getters, setters
    public Product() {}

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public double getPrice() { return price; }
    public void setPrice(double price) { this.price = price; }
    public int getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(int stockQuantity) { this.stockQuantity = stockQuantity; }
}

package com.example.serverless.model;

import com.amazonaws.services.dynamodbv2.datamodeling.*;

@DynamoDBTable(tableName = "Orders")
public class Order {
    @DynamoDBHashKey(attributeName = "id")
    private String id;
    
    @DynamoDBAttribute(attributeName = "customerId")
    private String customerId;
    
    @DynamoDBAttribute(attributeName = "productId")
    private String productId;
    
    @DynamoDBAttribute(attributeName = "quantity")
    private int quantity;
    
    @DynamoDBAttribute(attributeName = "status")
    private String status;

    // Constructors, getters, setters
    public Order() {}

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}
```

### AWS SAM Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless E-commerce API

Globals:
  Function:
    Runtime: java11
    MemorySize: 512
    Timeout: 30
    Environment:
      Variables:
        TABLE_NAME: !Ref ProductsTable

Resources:
  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      DefinitionBody:
        Paths:
          /products:
            get:
              x-amazon-apigateway-integration:
                uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${GetProductsFunction.Arn}/invocations
                httpMethod: POST
                type: aws_proxy
            post:
              x-amazon-apigateway-integration:
                uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${CreateProductFunction.Arn}/invocations
                httpMethod: POST
                type: aws_proxy
          /products/{id}:
            get:
              x-amazon-apigateway-integration:
                uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${GetProductFunction.Arn}/invocations
                httpMethod: POST
                type: aws_proxy

  GetProductsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.serverless.api.ProductHandler::getProducts
      CodeUri: target/serverless-app-1.0.jar
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /products
            Method: get
            RestApiId: !Ref ApiGateway

  CreateProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.serverless.api.ProductHandler::createProduct
      CodeUri: target/serverless-app-1.0.jar
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /products
            Method: post
            RestApiId: !Ref ApiGateway

  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: Products
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

Outputs:
  ApiUrl:
    Description: API Gateway URL
    Value: !Sub "https://${ApiGateway}.execute-api.${AWS::Region}.amazonaws.com/prod"
```

### Serverless Framework Configuration
```yaml
service: serverless-ecommerce

provider:
  name: aws
  runtime: java11
  stage: dev
  region: us-east-1

functions:
  getProducts:
    handler: com.example.serverless.api.ProductHandler::getProducts
    events:
      - http:
          path: products
          method: get

  createProduct:
    handler: com.example.serverless.api.ProductHandler::createProduct
    events:
      - http:
          path: products
          method: post

  getProduct:
    handler: com.example.serverless.api.ProductHandler::getProduct
    events:
      - http:
          path: products/{id}
          method: get

resources:
  Resources:
    ProductsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: Products
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
```

## Pros and Cons

### Advantages
✅ **No Server Management**: No infrastructure to provision or manage
✅ **Auto-scaling**: Automatically scales based on demand
✅ **Pay-per-Use**: Pay only for execution time
✅ **Faster Development**: Focus on code, not infrastructure
✅ **High Availability**: Built-in redundancy and availability
✅ **Reduced Operational Overhead**: Less DevOps work
✅ **Built-in Monitoring**: CloudWatch integration

### Disadvantages
❌ **Cold Starts**: First invocation latency
❌ **Vendor Lock-in**: Tied to specific cloud provider
❌ **Execution Limits**: Timeout and memory constraints
❌ **Debugging Challenges**: Harder to debug distributed functions
❌ **Statelessness**: Must manage state externally
❌ **Complexity**: Many moving pieces to understand
❌ **Cost Unpredictability**: Can be expensive at scale

## Real-World Use Cases

### Web Application Backend
- **Functions**: API endpoints, authentication, data processing
- **Benefits**: Auto-scaling, pay-per-use, reduced ops
- **Use Case**: REST API for mobile/web applications

### Data Processing Pipeline
- **Functions**: ETL, data transformation, analytics
- **Benefits**: Event-driven, scalable processing
- **Use Case**: Process uploaded files, generate reports

### IoT Backend
- **Functions**: Device data processing, alerting
- **Benefits**: Handle millions of devices, event-driven
- **Use Case**: Smart home, industrial IoT

### Chatbot
- **Functions**: Natural language processing, API integration
- **Benefits**: Scale with conversation volume
- **Use Case**: Customer service chatbots

## Interview Discussion Points

### Common Questions

**Q: How do you handle cold starts in serverless?**
A: Cold start mitigation strategies:
- Keep functions warm with scheduled pings
- Use provisioned concurrency (AWS)
- Optimize function initialization
- Use lighter runtime (Node.js vs Java)
- Choose smaller package size

**Q: How do you manage state in serverless applications?**
A: State management strategies:
- External databases (DynamoDB, RDS)
- Caching layers (Redis, ElastiCache)
- Object storage (S3)
- Session management with JWT
- Stateful services for complex state

**Q: How do you handle authentication in serverless?**
A: Authentication strategies:
- API Gateway with Cognito
- JWT validation in functions
- OAuth 2.0 / OpenID Connect
- Custom authorizers
- Third-party auth services (Auth0)

**Q: How do you monitor serverless applications?**
A: Monitoring strategies:
- CloudWatch for metrics and logs
- X-Ray for distributed tracing
- Custom metrics for business logic
- Alerting on errors and latency
- Third-party monitoring tools (Datadog, New Relic)

**Q: When would you choose serverless over containers?**
A: Choose serverless when:
- Workload is sporadic or unpredictable
- Want to minimize operational overhead
- Rapid prototyping or MVP
- Cost-sensitive with variable load
Choose containers when:
- Long-running processes
- Predictable, continuous workload
- Need more control over environment
- Vendor lock-in is a concern

### Advanced Topics

**Function Composition**
- Step Functions for orchestration
- Event-driven workflows
- Chaining functions
- Error handling and retries

**Performance Optimization**
- Memory vs CPU optimization
- Connection pooling
- Caching strategies
- Database optimization

**Security**
- Least privilege IAM roles
- VPC configuration
- Secrets management
- Input validation

## Best Practices

### Function Design
1. **Single responsibility**: One function, one purpose
2. **Stateless**: Don't maintain state in functions
3. **Idempotent**: Handle duplicate invocations safely
4. **Short-lived**: Keep execution time short
5. **Error handling**: Robust error handling and logging

### Performance
1. **Optimize cold starts**: Minimize initialization
2. **Right-size memory**: Balance memory and CPU
3. **Use caching**: Cache frequently accessed data
4. **Connection pooling**: Reuse database connections
5. **Async processing**: Use async for long operations

### Cost Optimization
1. **Monitor usage**: Track execution time and invocations
2. **Optimize duration**: Reduce function execution time
3. **Use provisioned concurrency**: For consistent performance
4. **Choose right services**: Use cost-effective services
5. **Set budgets**: Prevent unexpected costs

## Related Patterns

- **Event-Driven Architecture**: Natural fit with serverless
- **Microservices**: Serverless as microservices
- **API Gateway Pattern**: Centralized entry point
- **CQRS**: Separate read and write functions

## Technology Choices

### Cloud Providers
- **AWS Lambda**: Most mature serverless offering
- **Azure Functions**: Microsoft's serverless platform
- **Google Cloud Functions**: Google's serverless offering
- **Cloudflare Workers**: Edge computing functions

### Frameworks
- **Serverless Framework**: Multi-cloud deployment
- **AWS SAM**: AWS-specific serverless framework
- **Terraform**: Infrastructure as code
- **Pulumi**: Modern IaC with real languages

### Java Runtimes
- **Java 11/17**: Standard Java runtimes
- **Quarkus**: Fast startup, low memory
- **Micronaut**: Lightweight framework
- **Spring Native**: GraalVM native images

---

**Next**: Learn about [Domain-Driven Design](./domain-driven-design.md) for business-centric modeling
