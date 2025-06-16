# Cart Service

This microservice manages shopping carts for the e-commerce platform. It is built with Spring Boot and uses MongoDB for persistence. RabbitMQ is used for asynchronous messaging and the service queries the Product Service to validate items.

## Prerequisites
- Java 21
- Docker

## Running Locally
1. Start dependencies:
   ```bash
   docker-compose up -d
   ```
   MongoDB will be available on port `27017` and RabbitMQ on `5672` (`15672` for the management UI).
2. Launch the service:
   ```bash
   ./mvnw spring-boot:run
   ```
   Configuration values can be overridden in `src/main/resources/application.properties` or via environment variables.

### Key Environment Variables
- `CART_SERVICE_PORT` – port to run the service (default `8084`)
- `SPRING_DATA_MONGODB_URI` – MongoDB connection string
- `SPRING_RABBITMQ_HOST` / `SPRING_RABBITMQ_PORT` / `SPRING_RABBITMQ_USERNAME` / `SPRING_RABBITMQ_PASSWORD`
- `PRODUCT_SERVICE_URL` – base URL of the Product Service
- `AUTH_SERVICE_URL` – base URL of the Auth Service

## Running Tests
```bash
./mvnw test
```

## Docker Image
To build and run the container manually:
```bash
docker build -t cart-service .
docker run -p 8084:8084 -e CART_SERVICE_PORT=8084 cart-service
```
The provided `docker-compose.yml` also defines a service entry for the container.

## API Overview
Base path: `/api/carts` (also accessible via `/api/cart`).

### Add item
`POST /{userId}/items`
```json
{
  "productId": 1,
  "quantity": 2
}
```
Adds the given product to the user's cart.

### View cart
`GET /{userId}` – retrieves the cart for the given user.

### Update quantity
`PUT /{cartIdentifier}/items/{productId}` with body `{ "quantity": 3 }` – updates or removes an item.

### Remove item
`DELETE /{cartIdentifier}/items/{productId}` – removes an item from the cart.

## Data Model
```java
public class Cart {
    private String id;
    private String cartIdentifier;
    private List<CartItem> items;
    private LocalDateTime lastModifiedDate;
}

public class CartItem {
    private Long productId;
    private int quantity;
    private BigDecimal price;
}
```

## Messaging
The service publishes `ItemAddedToCartEvent` to `cartExchange` with routing key `cart.item.added`. It listens to `order.created.queue` and `product.updated.queue` to clear carts when orders are placed and to update item prices when products change.

## External Interactions
When adding items, product details are fetched from `${PRODUCT_SERVICE_URL}/api/products/{productId}`.
