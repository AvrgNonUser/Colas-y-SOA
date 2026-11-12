# BikeStore Async v1.0

Procesamiento asíncrono de pedidos con **Spring Boot 3 + RabbitMQ**.

```
Front-end ──POST /api/ordenar──▶ OrderProducer ──order.created──▶ [bikestore.orders]
                                                                       │
                                                         bikestore.payment.queue
                                                                       ▼
                                                              PaymentWorker (50 % fallo)
                                                       ┌───────────────┴─────────────────┐
                                           éxito: order.paid                 3 intentos fallidos
                                                       ▼                                  ▼
                                          bikestore.email.queue              bikestore.payment.dlq (DLQ)
                                                       ▼
                                                 EmailWorker (solo si PAID)
```

## Requisitos
- Java 17+
- Maven 3.9+
- Docker (para RabbitMQ)

## Ejecutar
```bash
docker compose up -d          # RabbitMQ (consola: http://localhost:15672  usuario bikestore / bikestore123)
mvn spring-boot:run
```

## Probar
```bash
curl -X POST http://localhost:8080/api/ordenar \
  -H "Content-Type: application/json" \
  -d '{
        "customerName": "Ana Pérez",
        "customerEmail": "ana@example.com",
        "items": [
          {"sku": "BIKE-MTB-29", "quantity": 1, "unitPrice": 850.00},
          {"sku": "HELMET-L",    "quantity": 2, "unitPrice": 35.50}
        ]
      }'
```
Respuesta: `202 Accepted` con el `pedidoId`.

Para ver la DLQ en acción, envíe varios pedidos seguidos (con 50 % de fallo por intento, ~12,5 % acabará en la DLQ):
```bash
for i in $(seq 1 20); do
  curl -s -X POST http://localhost:8080/api/ordenar -H "Content-Type: application/json" \
    -d '{"customerName":"Test","customerEmail":"t@example.com","items":[{"sku":"X","quantity":1,"unitPrice":10}]}' > /dev/null
done
```
Luego abra `http://localhost:15672` → Queues → `bikestore.payment.dlq` → *Get messages* para analizarlos offline.

## Ejemplo de log
```
[OrderProducer] pedidoId=PED-3F9A1C2B timestamp=2026-11-11T15:02:01.120Z thread=http-nio-8080-exec-1 | pedido recibido, total=921.00, publicando order.created
[PaymentWorker] pedidoId=PED-3F9A1C2B timestamp=... thread=org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0-1 | procesando cobro de 921.00 (intento 1)
[PaymentWorker] pedidoId=PED-3F9A1C2B timestamp=... | cobro RECHAZADO en intento 1
[PaymentWorker] pedidoId=PED-3F9A1C2B timestamp=... | procesando cobro de 921.00 (intento 2)
[PaymentWorker] pedidoId=PED-3F9A1C2B timestamp=... | cobro EXITOSO, paymentStatus=PAID
[EmailWorker]   pedidoId=PED-3F9A1C2B timestamp=... thread=...#1-1 | e-mail enviado a ana@example.com ...
[DLQ]           pedidoId=PED-77B0E4D1 timestamp=... | reintentos agotados, enviando a bikestore.payment.dlq
```

## Configuración útil (`application.yml`)
| Propiedad | Descripción |
|---|---|
| `bikestore.payment.failure-rate` | Probabilidad de fallo por intento (0.5 = 50 %). Use `1.0` para forzar la DLQ o `0.0` para nunca fallar. |
| `spring.rabbitmq.listener.simple.retry.max-attempts` | Intentos totales (3). Cámbielo a 4 si interpreta «3 reintentos» como 1 intento + 3 reintentos. |
| `spring.rabbitmq.listener.simple.concurrency` | Hilos consumidores por cola. |

## Estructura
```
config/    RabbitConfig (exchanges, colas, DLX/DLQ, JSON, recoverer), AmqpNames
producer/  OrderProducerController (API Ordenar) + OrderProducer
worker/    PaymentWorker, EmailWorker
model/     OrderRequest, OrderMessage, PaymentProcessedEvent, ...
util/      EventLogger (pedidoId + timestamp + thread)
```
