# Cart Service

The Cart Service manages customer shopping carts in the e‑commerce system. It is built with Spring Boot, stores data in MongoDB and communicates with other services via RabbitMQ and REST APIs.

## Prerequisites
- Java 21
- Maven
- Docker and Docker Compose

## Running with Docker Compose
From the repository root run:

```bash
docker-compose up -d rabbitmq cart-mongodb cart-service
```

This starts RabbitMQ (`5672`/`15672`), MongoDB (`27017`) and the Cart Service on port `8084`.

### Local Development
You can also run the application directly:

```bash
cd cart-service
./mvnw spring-boot:run
```

## Configuration
Default values are defined in `src/main/resources/application.properties` and can be overridden with environment variables.

| Variable | Default | Description |
|----------|---------|-------------|
| `CART_SERVICE_PORT` | `8084` | HTTP port for the service |
| `SPRING_DATA_MONGODB_URI` | `mongodb://localhost:27017/cartdb` | MongoDB connection URI |
| `SPRING_RABBITMQ_HOST` | `localhost` | RabbitMQ host |
| `SPRING_RABBITMQ_PORT` | `5672` | RabbitMQ port |
| `SPRING_RABBITMQ_USERNAME` | `guest` | RabbitMQ username |
| `SPRING_RABBITMQ_PASSWORD` | `guest` | RabbitMQ password |
| `PRODUCT_SERVICE_URL` | `http://localhost:8083` | Base URL of the Product Service |

## REST API
All routes are served under `/api/carts` (also accessible as `/api/cart`). Examples below use `/api/carts`.

### Add item
`POST /api/carts/{userId}/items`

Request body:
```json
{
  "productId": 1,
  "quantity": 2
}
```
Adds a product to the user’s cart and returns the updated `Cart` document.

### View cart
`GET /api/carts/{userId}`

Returns a `CartResponseDTO` describing the user’s current items.

### Update quantity
`PUT /api/carts/{cartIdentifier}/items/{productId}`

Request body:
```json
{
  "quantity": 3
}
```
Sets the quantity of the given product. If the quantity becomes zero the item is removed and the cart is deleted when empty.

### Remove item
`DELETE /api/carts/{cartIdentifier}/items/{productId}`

Removes the product from the cart. The cart itself is deleted if it becomes empty.

## Events
The service publishes and consumes events via RabbitMQ.

### `ItemAddedToCartEvent`
Published whenever an item is added.
- **Exchange:** `cartExchange`
- **Routing key:** `cart.item.added`
```json
{
  "cartIdentifier": "1",
  "productId": 2,
  "quantityAdded": 1
}
```

### `OrderCreatedEvent`
Consumed from the `order.created.queue`. When received the cart for the user is cleared.
```json
{
  "orderId": 10,
  "userId": 1,
  "totalAmount": 20.00,
  "items": []
}
```

### `ProductUpdatedEvent`
Consumed from the `product.updated.queue`. Updates item prices when a product changes.
```json
{
  "productId": 5,
  "newName": "name",
  "newPrice": 9.99,
  "newStock": 100
}
```

## Running Tests
Execute unit tests with:
```bash
./mvnw test
```

## Building a Docker Image
You can build the container yourself:
```bash
cd cart-service
docker build -t cart-service .
docker run -p 8084:8084 cart-service
```
Environment variables from the table above can be supplied with `-e` options.
