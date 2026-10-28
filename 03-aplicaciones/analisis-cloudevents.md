# analisis-cloudevents.md
# Completa este archivo con tus respuestas al TODO 4 de la Fase 3.

## TODO 4: Análisis de Trazabilidad (CloudEvents)

```
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-publisher -c order-publisher
2026-10-24 02:33:33,680 [INFO] Esperando 5 segundos para que el sidecar de Dapr esté listo...
2026-10-24 02:33:38,688 [INFO] ✅ Orden publicada — orderId=1001  status=created
2026-10-24 02:33:39,697 [INFO] ✅ Orden publicada — orderId=1002  status=processing
2026-10-24 02:33:40,704 [INFO] ✅ Orden publicada — orderId=1003  status=shipped
2026-10-24 02:33:41,707 [INFO] Publicación completada.
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-subscriber -c order-subscriber -f
2026-10-24 02:33:39,697 [INFO] 127.0.0.1 - - [24/Oct/2026 02:33:39] "POST /orders HTTP/1.1" 200 -
2026-10-24 02:33:40,705 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-24 02:33:40,705 [INFO] 📨 CloudEvent recibido
2026-10-24 02:33:40,705 [INFO]    id          : 4b36facd-21c8-4872-bac3-1b31b9076953
2026-10-24 02:33:40,705 [INFO]    source      : order-publisher
2026-10-24 02:33:40,705 [INFO]    type        : com.dapr.event.sent
2026-10-24 02:33:40,705 [INFO]    traceparent : 
2026-10-24 02:33:40,705 [INFO]    data        : {'orderId': '1003', 'status': 'shipped'}
2026-10-24 02:33:40,705 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-24 02:33:40,706 [INFO] 127.0.0.1 - - [24/Oct/2026 02:33:40] "POST /orders HTTP/1.1" 200 -
```

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

Lo genera el sidecar de Dapr del publisher (`daprd`) cuando recibe la llamada a `/v1.0/publish/...`. Es un UUID único por evento (en los logs: `4b36facd-21c8-4872-bac3-1b31b9076953`). El código Python del publisher no es responsable de crear este id. 

La razón por la cuál es importante para la idempotencia es porque los brokers suelen entregar al menos una vez. Si hay un reintento o una reentrega, el subscriber puede recibir el mismo evento dos veces. Como el `id` es el mismo en cada copia, el subscriber puede guardar los `id` ya procesados y descartar los repetidos (por ejemplo, para no cobrar dos veces una orden).

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

Tiene el valor `order-publisher`. Dapr lo toma de la anotación **`dapr.io/app-id`** del Pod del publisher (en `publisher-job.yaml`), no del código ni del nombre del Pod. De esta manera se identifica qué aplicación produjo el evento.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

Implementa el estándar W3C Trace Context. Es una cabecera con el formato `versión-traceId-spanId-flags`, que permite seguir una misma petición a través de varios servicios.

OpenTelemetry usa W3C Trace Context como formato por defecto para propagar el contexto entre servicios. Dapr genera y propaga el `traceparent`, y exporta las trazas en formato compatible con OpenTelemetry (OTLP o Zipkin). Así, el publisher, el broker y el subscriber aparecen como spans de una misma traza en un backend como Grafana.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

| Atributo | Por qué sirve a auditoría |
|---|---|
| `id` | Identifica cada evento de forma única. Permite contar eventos, detectar duplicados y comprobar que ninguno falta. |
| `source` | Dice qué aplicación emitió el evento y permite filtrar por servicio. |
| `type` | Dice qué tipo de evento ocurrió, para agrupar o buscar por tipo. |
| `time` | Fecha y hora de emisión. Es la base para consultar "las últimas 24 h" y ordenar los eventos. |
| `traceparent` | Une los eventos de una misma operación entre servicios y reconstruye el recorrido completo. |
| `data` | Contiene el dato de negocio (`orderId`, `status`) con el que se verifica qué pasó con cada orden. |

Los más importantes son `id`, `source`, `time` y `traceparent`. Responden a quién emitió el evento, cuándo, si es único y con qué otros eventos se relaciona.

En el evento recibido por el subscriber no aparece `time` en los logs porque el código solo imprime algunos atributos. El sobre CloudEvents que envía Dapr sí lo incluye.
