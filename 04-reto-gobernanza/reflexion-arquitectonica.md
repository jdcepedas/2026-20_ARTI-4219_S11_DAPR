# reflexion-arquitectonica.md
# Completa este archivo con tus respuestas al Reto 3 de la Fase 4.

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

Para la primera parte de esta prueba se definió una configuración de Dapr ttlInSeconds

```
- name: ttlInSeconds
  value: "120"
```
Según la documentación esto permite que se aplique un campo expiration a los CloudEvents de manera que puedan ser descartados. 
```
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl get components.dapr.io order-pubsub -o yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"dapr.io/v1alpha1","kind":"Component","metadata":{"annotations":{},"name":"order-pubsub","namespace":"default"},"spec":{"metadata":[{"name":"redisHost","value":"redis-master.redis.svc.cluster.local:6379"},{"name":"redisPassword","value":""},{"name":"ttlInSeconds","value":"120"}],"type":"pubsub.redis","version":"v1"}}
  creationTimestamp: "2026-10-24T02:09:02Z"
  generation: 6
  name: order-pubsub
  namespace: default
  resourceVersion: "138635"
  uid: 9ec2baf2-c78e-4086-bab9-791b21aff984
spec:
  metadata:
  - name: redisHost
    value: redis-master.redis.svc.cluster.local:6379
  - name: redisPassword
    value: ""
  - name: ttlInSeconds
    value: "120"
  type: pubsub.redis
  version: v1
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=0
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl delete job order-publisher --ignore-not-found
job.batch "order-publisher" deleted
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 03-aplicaciones/publisher-job.yaml
job.batch/order-publisher created
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % sleep 130
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=1
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-subscriber -c order-subscriber -f
2026-10-28 00:47:55,392 [INFO] 127.0.0.1 - - [28/Oct/2026 00:47:55] "POST /orders HTTP/1.1" 200 -
2026-10-28 00:47:55,392 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-28 00:47:55,392 [INFO] 📨 CloudEvent recibido
2026-10-28 00:47:55,392 [INFO]    id          : ab276587-1802-44c7-a239-cac5296ff047
2026-10-28 00:47:55,392 [INFO]    source      : order-publisher
2026-10-28 00:47:55,392 [INFO]    type        : com.dapr.event.sent
2026-10-28 00:47:55,392 [INFO]    traceparent : 
2026-10-28 00:47:55,392 [INFO]    data        : {'orderId': '1002', 'status': 'processing'}
2026-10-28 00:47:55,392 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-28 00:47:55,393 [INFO] 127.0.0.1 - - [28/Oct/2026 00:47:55] "POST /orders HTTP/1.1" 200 -
```

Sin embargo, en la salida de los comandos se puede evidenciar que esta configuración no funciona para Redis dado que el CloudEvent no agrega es campo de expiración que se pretende. Consultando la documentación de Redis se encuentra lo siguiente:

Lo único que se puede hacer desde la configuración es streamTTL. Es retención del broker, es aproximada y depende de que se publiquen mensajes nuevos.

De manera que se aplica esta configuración:

```
    - name: streamTTL
      value: "2m"
```

Sin embargo al correr los comandos se encuentra un problema similar:

```
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 04-reto-gobernanza/pubsub-redis-challenge.yaml
component.dapr.io/order-pubsub configured
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl get components.dapr.io order-pubsub -o yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"dapr.io/v1alpha1","kind":"Component","metadata":{"annotations":{},"name":"order-pubsub","namespace":"default"},"spec":{"metadata":[{"name":"redisHost","value":"redis-master.redis.svc.cluster.local:6379"},{"name":"redisPassword","value":""},{"name":"streamTTL","value":"2m"}],"type":"pubsub.redis","version":"v1"}}
  creationTimestamp: "2026-10-24T02:09:02Z"
  generation: 7
  name: order-pubsub
  namespace: default
  resourceVersion: "139748"
  uid: 9ec2baf2-c78e-4086-bab9-791b21aff984
spec:
  metadata:
  - name: redisHost
    value: redis-master.redis.svc.cluster.local:6379
  - name: redisPassword
    value: ""
  - name: streamTTL
    value: 2m
  type: pubsub.redis
  version: v1
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=0
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl delete job order-publisher --ignore-not-found
kubectl apply -f 03-aplicaciones/publisher-job.yaml
job.batch "order-publisher" deleted
job.batch/order-publisher created
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % sleep 130
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=1
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-subscriber -c order-subscriber -f
2026-10-28 00:55:49,730 [INFO] 127.0.0.1 - - [28/Oct/2026 00:55:49] "POST /orders HTTP/1.1" 200 -
2026-10-28 00:55:49,730 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-28 00:55:49,730 [INFO] 📨 CloudEvent recibido
2026-10-28 00:55:49,730 [INFO]    id          : adff2cb0-f22d-41b0-9d5b-66f2913b998d
2026-10-28 00:55:49,730 [INFO]    source      : order-publisher
2026-10-28 00:55:49,731 [INFO]    type        : com.dapr.event.sent
2026-10-28 00:55:49,731 [INFO]    traceparent : 
2026-10-28 00:55:49,731 [INFO]    data        : {'orderId': '1001', 'status': 'created'}
2026-10-28 00:55:49,731 [INFO] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2026-10-28 00:55:49,731 [INFO] 127.0.0.1 - - [28/Oct/2026 00:55:49] "POST /orders HTTP/1.1" 200 -

```

Aun configurado este nuevo parámetro para redis los mensajes siguen llegando todos los mensajes.
Con Redis no se puede seguir el taller al pie de la letra usando solo la configuración. streamTTL solo borra mensajes cuando llega una publicación nueva. Si después de esperar nadie publica, los 3 mensajes siguen en Redis y el subscriber los recibe igual.

Otra de las pruebas que se pueden realizar para evidenciar el comportamiento es enviar dos tandas de eventos:
```
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli CONFIG SET stream-node-max-entries 1
OK
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 04-reto-gobernanza/pubsub-redis-challenge.yaml
component.dapr.io/order-pubsub unchanged
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=0
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XTRIM orders MAXLEN 0
3
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XLEN orders
0
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl delete job order-publisher --ignore-not-found
job.batch "order-publisher" deleted
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 03-aplicaciones/publisher-job.yaml
job.batch/order-publisher created
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XLEN orders
3
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XRANGE orders - +
1793149320176-0
data
{"data":{"orderId":"1001","status":"created"},"datacontenttype":"application/json","id":"5bb85f10-a921-44cb-b0d4-cbbafd151415","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:02:00Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
metadata
{}
1793149321186-0
data
{"data":{"orderId":"1002","status":"processing"},"datacontenttype":"application/json","id":"20139bf2-7d7d-4ee7-af94-c32f8c6227d1","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:02:01Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
metadata
{}
1793149322194-0
data
{"data":{"orderId":"1003","status":"shipped"},"datacontenttype":"application/json","id":"7679974a-f34f-4009-a149-5c0cb6c6d2c3","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:02:02Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
metadata
{}
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % sleep 130
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl delete job order-publisher
job.batch "order-publisher" deleted
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 03-aplicaciones/publisher-job.yaml
job.batch/order-publisher created
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XLEN orders
3
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n redis redis-master-0 -- redis-cli XRANGE orders - +
1793149561881-0
metadata
{}
data
{"data":{"orderId":"1001","status":"created"},"datacontenttype":"application/json","id":"2a666e49-2f96-4b67-bcd5-535345863566","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:06:01Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
1793149562885-0
data
{"data":{"orderId":"1002","status":"processing"},"datacontenttype":"application/json","id":"25a0de1b-73f6-485e-95b6-6ab186b6818c","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:06:02Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
metadata
{}
1793149563889-0
data
{"data":{"orderId":"1003","status":"shipped"},"datacontenttype":"application/json","id":"a4e8c56b-c706-4236-a737-8c264d7a6000","pubsubname":"order-pubsub","source":"order-publisher","specversion":"1.0","time":"2026-10-28T01:06:03Z","topic":"orders","traceid":"","traceparent":"","tracestate":"","type":"com.dapr.event.sent"}
metadata
{}
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=1
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-subscriber -c order-subscriber --tail=-1 | grep "id  "
2026-10-28 01:06:49,137 [INFO]    id          : a4e8c56b-c706-4236-a737-8c264d7a6000
2026-10-28 01:06:49,138 [INFO]    id          : 2a666e49-2f96-4b67-bcd5-535345863566
2026-10-28 01:06:49,139 [INFO]    id          : 25a0de1b-73f6-485e-95b6-6ab186b6818c
```
En este caso se puede observar que los eventos de la primera tanda expiran, pero los de la segunda tanda sí son recibidos. De manera que se necesita una segunda publicación para lograr este efecto.

#### Prueba con RabbitMq

Para evidenciar que la expiración sí se realiza con otra tecnología como RabbitMq se crearon dos nuevos manifiestos 02-componentes-declarativos/rabbitmq.yaml y 04-reto-gobernanza/pubsub-rabbitmq-challenge.yaml con ls configuraciones correspondientes. Una vez hecho esto realizamos de nuevo la prueba del taller y podemos comprobar que en este caso RabbitMq sí acepta ttlInSeconds.

```
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=0
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl delete job order-publisher --ignore-not-found
job.batch "order-publisher" deleted
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl apply -f 03-aplicaciones/publisher-job.yaml
job.batch/order-publisher created
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % sleep 130
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl exec -n rabbitmq deploy/rabbitmq -- rabbitmqctl list_queues name messages
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
order-subscriber-orders	0
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl scale deployment order-subscriber --replicas=1
deployment.apps/order-subscriber scaled
jcepeda@Juans-MacBook-Pro 2026-20_ARTI-4219_S11_DAPR % kubectl logs -l app=order-subscriber -c order-subscriber --tail=-1 -f
 * Serving Flask app 'app'
 * Debug mode: off
2026-10-28 01:18:16,056 [INFO] WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5001
 * Running on http://10.244.0.44:5001
2026-10-28 01:18:16,056 [INFO] Press CTRL+C to quit
2026-10-28 01:18:16,091 [INFO] 127.0.0.1 - - [28/Oct/2026 01:18:16] "GET /dapr/config HTTP/1.1" 404 -
2026-10-28 01:18:16,091 [INFO] Dapr solicitó las suscripciones: [{'pubsubname': 'order-pubsub', 'topic': 'orders', 'route': '/orders'}]
2026-10-28 01:18:16,092 [INFO] 127.0.0.1 - - [28/Oct/2026 01:18:16] "GET /dapr/subscribe HTTP/1.1" 200 -

```

Para volver a Redis utilizamos los siguientes comandos: 

```
kubectl apply -f 04-reto-gobernanza/pubsub-redis-challenge.yaml

kubectl rollout restart deployment order-subscriber

kubectl delete -f 02-componentes-declarativos/rabbitmq.yaml

```

### 2. El patrón sidecar como frontera de responsabilidad

> _Tu respuesta aquí_

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

No. Dapr unifica la API, pero cada broker se comporta distinto. Un caso concreto es el **TTL**:

- En Dapr, el TTL exacto se define **por mensaje**, con `metadata.ttlInSeconds` en la petición de publicación (decisión del publisher).
- Probé `ttlInSeconds: "120"` en el Component de Redis y **no tuvo efecto**: los mensajes en el stream no traían el atributo `expiration` y se entregaron más de 14 horas después de publicarse.
- Para Redis Streams, la opción declarativa es `streamTTL`, que elimina entradas viejas del stream. Es **aproximada** y se aplica cuando llega un mensaje nuevo.
- Probé `streamTTL: "2m"`: publiqué 3 mensajes, esperé más de 2 minutos, publiqué 3 más y el stream seguía con 6 entradas, y el subscriber recibió mensajes de la primera tanda. En una prueba aparte en Redis, un recorte aproximado (`MINID ~`) no eliminó nada en un stream pequeño, mientras que el recorte exacto sí. Por eso `streamTTL` no sirve como TTL de entrega en este laboratorio.
- En Azure Service Bus, en cambio, el TTL lo maneja el propio broker de forma nativa.

Por eso, migrar de Redis a otro broker exige revisar cómo se aplica el TTL y los consumer groups, aunque el código Python no cambie.
