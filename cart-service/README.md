# Cart Service

The Cart Service manages shopping carts for the e-commerce platform. It is a Spring Boot application that stores cart data in MongoDB and communicates with other services via RabbitMQ and REST calls.

## Prerequisites

- Java 21
- Maven
- Docker with Docker Compose

## Running the Service

The easiest way to run the service together with MongoDB and RabbitMQ is via the root `docker-compose.yml` file:

```bash
# From the repository root
docker-compose up -d rabbitmq cart-mongodb cart-service
```

This starts RabbitMQ (ports `5672` and `15672`), a MongoDB instance (port `27017`) and the Cart Service itself on port `8084`.

During development you can also run the application directly using Maven:

```bash
cd cart-service
./mvnw spring-boot:run
```

## Running Tests

Unit tests can be executed with Maven:

```bash
./mvnw test
```

## Configuration

Default settings are defined in `src/main/resources/application.properties` and can be overridden with environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `CART_SERVICE_PORT` | `8084` | HTTP port for the service |
| `SPRING_DATA_MONGODB_URI` | `mongodb://localhost:27017/cartdb` | MongoDB connection URI |
| `SPRING_RABBITMQ_HOST` | `localhost` | RabbitMQ host |
| `SPRING_RABBITMQ_PORT` | `5672` | RabbitMQ port |
| `SPRING_RABBITMQ_USERNAME` | `guest` | RabbitMQ username |
| `SPRING_RABBITMQ_PASSWORD` | `guest` | RabbitMQ password |
| `PRODUCT_SERVICE_URL` | `http://localhost:8083` | Base URL of the Product Service |

## API Endpoints

All endpoints are served under `/api/cart` and require an `X-User-Id` header that identifies the authenticated user.

### Headers

All requests must include:

```
X-User-Id: <user id>
```

Replace `<user id>` with the identifier of the user making the request.

### Add item

`POST /api/cart/items`

Adds a product to the authenticated user's cart. Creates the cart if it does not exist.

Example request:

```bash
curl -X POST http://localhost:8084/api/cart/items \
  -H "X-User-Id: 1" \
  -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":2}'
```

Example response:

```json
{
  "id": "abc123",
  "cartIdentifier": "1",
  "items": [
    { "productId": 1, "quantity": 2, "price": 9.99 }
  ],
  "lastModifiedDate": "2025-07-01T10:00:00"
}
```

Returns the updated `Cart` document.

### View cart

`GET /api/cart`

Example request:

```bash
curl -H "X-User-Id: 1" http://localhost:8084/api/cart
```

Example response:

```json
{
  "userId": 1,
  "items": [
    { "productId": 1, "quantity": 2, "price": 9.99 }
  ]
}
```

Returns a `CartResponseDTO` with the current items for the user.

### Update quantity

`PUT /api/cart/items/{productId}`

Updates the quantity of a product in the authenticated user's cart.

Example request:

```bash
curl -X PUT http://localhost:8084/api/cart/items/1 \
  -H "X-User-Id: 1" \
  -H "Content-Type: application/json" \
  -d '{"quantity":3}'
```

Example response:

```json
{
  "id": "abc123",
  "cartIdentifier": "1",
  "items": [
    { "productId": 1, "quantity": 3, "price": 9.99 }
  ],
  "lastModifiedDate": "2025-07-01T10:05:00"
}
```

If the quantity becomes zero the item is removed. When the last item is removed the cart document is deleted.

### Remove item

`DELETE /api/cart/items/{productId}`

Removes the given product from the cart.

Example request:

```bash
curl -X DELETE http://localhost:8084/api/cart/items/1 \
  -H "X-User-Id: 1"
```

Example response:

```json
{
  "id": "abc123",
  "cartIdentifier": "1",
  "items": [],
  "lastModifiedDate": "2025-07-01T10:06:00"
}
```

The cart is deleted if no items remain.

## Event Flow

The service interacts with RabbitMQ to notify other services and react to updates.

### ItemAddedToCartEvent

Published whenever an item is added to a cart.

- **Exchange:** `cartExchange`
- **Routing key:** `cart.item.added`

Payload example:

```json
{
  "cartIdentifier": "1",
  "productId": 2,
  "quantityAdded": 1
}
```

### OrderCreatedEvent

Consumed from the `order.created.queue`. When received the cart for the given user is cleared.

Payload example:

```json
{
  "orderId": 10,
  "userId": 1,
  "totalAmount": 20.00,
  "items": []
}
```

### ProductUpdatedEvent

Consumed from the `product.updated.queue`. Updates cart item prices when a product's price changes.

Payload example:

```json
{
  "productId": 5,
  "newName": "name",
  "newPrice": 9.99,
  "newStock": 100
}
```

## Building a Docker Image

A Dockerfile is provided. To build and run the container directly:

```bash
cd cart-service
docker build -t cart-service .
docker run -p 8084:8084 cart-service
```

The same environment variables shown above can be passed with `-e` options if customization is required.

