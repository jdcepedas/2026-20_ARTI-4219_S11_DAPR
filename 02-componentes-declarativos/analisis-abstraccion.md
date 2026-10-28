# analisis-abstraccion.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 2.

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

Solo cambiaría **un archivo**: `02-componentes-declarativos/pubsub-redis.yaml` (el `Component`).

- Se cambia `spec.type` de `pubsub.redis` a `pubsub.kafka`.
- Se reemplazan los campos de `metadata` (`redisHost`, `redisPassword`) por los de Kafka (por ejemplo `brokers`, `authType`).
- Opcional: renombrar el archivo para que no diga "redis". Si se cambia `metadata.name` (`order-pubsub`), habría que actualizarlo también en el código, así que lo mejor es dejarlo igual.

**El código Python no cambia**, ni el del publisher ni el del subscriber. Los dos solo hablan con el sidecar de Dapr:

- El publisher llama a `http://localhost:3500/v1.0/publish/order-pubsub/orders`.
- El subscriber declara `pubsubname: "order-pubsub"` en `/dapr/subscribe`.

Ninguno conoce a Redis ni usa una librería de Redis o de Kafka. Tampoco cambian los Deployments ni el Job. Después de aplicar el YAML nuevo, solo hay que reiniciar los pods para que los sidecars carguen el componente: `kubectl rollout restart deployment order-subscriber`.

### 2. Inventario de 5 brokers soportados por Dapr

| Broker | `spec.type` en el YAML |
|---|---|
| Redis | `pubsub.redis` |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Apache Pulsar | `pubsub.pulsar` |
| AWS SNS/SQS | `pubsub.aws.snssqs` |
| Azure Service Bus Topics | `pubsub.azure.servicebus.topics` |


El campo que cambia es `spec.type`. También cambia el contenido de `spec.metadata`, porque cada broker tiene sus propios parámetros (host, credenciales, región, etc.). El resto del YAML (`apiVersion`, `kind`, `metadata.name`) se queda igual.

### 3. Manejo seguro de secretos en producción

Dapr ofrece el **building block de Secrets** con componentes **Secret Store** (`secretstores.*`). En vez de escribir el valor en el YAML, se le dice al componente que lo lea desde un almacén de secretos con `secretKeyRef`:

```yaml
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisPassword
      secretKeyRef:
        name: redis-secret   # nombre del secreto
        key: redis-password  # clave dentro del secreto
auth:
  secretStore: kubernetes    # componente Secret Store que se usa
```

- El YAML que va a Git solo contiene la referencia no la contraseña.
- El sidecar lee el valor real del Secret Store en tiempo de ejecución.
- En Kubernetes, el Secret Store `kubernetes` viene incluido y lee los `Secret` nativos del clúster.
- En producción se suelen usar almacenes externos, como HashiCorp Vault, Azure Key Vault, AWS Secrets Manager o GCP Secret Manager. Cada uno es un componente `secretstores.*`.

