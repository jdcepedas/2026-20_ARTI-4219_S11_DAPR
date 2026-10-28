# analisis-control-plane.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 1.

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

Corriendo el siguiente comando podemos verificar el tipo de objeto que es el dapr-sidecar-injector:

```
kubectl get mutatingwebhookconfiguration dapr-sidecar-injector -o yaml
```

El pod dapr-sidecar-injector es un Pod gestionado por un Deployment que ejecuta un servidor HTTPS de admisión. Se registra en el API Server mediante un MutatingWebhookConfiguration. Tiene que ser mutating porque su trabajo es modificar el spec del Pod para agregar el contenedor daprd, y eso solo se puede hacer durante la admisión, antes de que el Pod se guarde, porque después los contenedores son inmutables.

### 2. Réplicas del `dapr-operator` y comparación con Istio

Se encuentra corriendo solo una réplica del dapr-operator.

´´´
NAME                                     READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
dapr-operator-7c8bb589b7-rkzzs           1/1     Running   0          24h   10.244.0.6    dapr-lab-control-plane   <none>           <none>
´´´

El plano de control de Dapr es más liviano porque no intercepta todo el tráfico de red.
En cambio, Istio inyecta un proxy Envoy y usa reglas de iptables para capturar todo el tráfico entrante y saliente del pod, sea cual sea la aplicación, todo esto consume recursos como CPU y memoria, lo que crece con el número de servicios y pods.
En el caso de Dapr, este se considera un sidecar explícito, la aplicación llama a localhost:3500 (HTTP) o :50001 (gRPC) cuando quiere usar un building block (pub/sub, state, invoke). No hay captura transparente de tráfico. El operator solo vigila CRDs (Component, Configuration, Subscription, Resiliency) y se las entrega a los sidecars

### 3. Flujo de inyección del sidecar

El flujo de inyección del sidecar se utiliza para asociar efectivamente un sidecar a un deployment en particular. En el contexto del taller, habrán aplicaciones con lógica de negocio de publicación y de suscripción que tendrán un yaml con la definición del Deployment. Es en este archivo que se incluyen las anotaciones de Dapr.

#### Flujo 
 1. Se realiza el commando de apply kubectl apply -f deployment.yaml con las anotaciones correspondientes 
 2. El plano de control de Kubernetes o API Server dapr-lab-control-plane procesa el deployment y lo guarda en etcd
 3. El controlador de Deployments crea un ReplicaSet y un pod
 4. El API Server mira la lista de MutatingWebhookConfiguration y encuentra que dapr-sidecar-injector coincide con la petición de CREATE pods. Envía un AdmissionReview al Injector 
 5. El injecto revisa las anotaciones del pod para decidir si actúa
 6. El injector construye un parche al spec del pod con variables de Dapr
 7. El API Server aplica el parche y guarda el pod en etcd.
 8. El scheduler asigna el pod a un nodo y se arrancan los contenedores
 9. Dapr comienza a funcionar y se conecta al plano de control y abre el puerto de la aplicación
 10. El pod queda 2/2 running cuando la aplicación y daprd pasan los probes


---

## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

Los CRDs instalados por Dapr se pueden ver aquí:


```
kubectl get crds | grep dapr.io

components.dapr.io
configurations.dapr.io
httpendpoints.dapr.io 
mcpservers.dapr.io
resiliencies.dapr.io
subscriptions.dapr.io
workflowaccesspolicies.dapr.io
```

#### components.dapr.io
Declara un componente concreto (Redis, Kafka, Postgres…) que implementa un building block: pub/sub, state store, bindings, secrets.

#### configurations.dapr.io

Configura el comportamiento del runtime: tracing, mTLS, métricas, middleware, feature flags.

#### subscriptions.dapr.io
Suscripción declarativa: enruta los mensajes de un topic a un endpoint de la app, sin código de suscripción.

#### resiliencies.dapr.io

Define políticas de resiliencia: reintentos, timeouts y circuit breakers.

#### httpendpoints.dapr.io
Registra servicios HTTP externos para invocarlos con service invocation.



### 5. Relación entre `components.dapr.io` y el concepto de Building Block

Building block = la abstracción. Es la API estable que usa el código, por ejemplo POST localhost:3500/v1.0/publish/<pubsub-name>/<topic>. Es la misma sin importar qué hay detrás.

Component = la implementación. Es un recurso YAML que dice qué tecnología concreta cumple ese building block, con su configuración. Por ejemplo, en pubsub-redis.yaml se usa type: pubsub.redis.

Una app con lógica de negocio solo conoce el nombre del componente (metadata.name). Si se cambia pubsub.redis por pubsub.kafka en el YAML y se mantiene el nombre, el código no cambia. Esa es la portabilidad que ofrece Dapr.