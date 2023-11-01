# Access the Kafka Cluster by Envoy Gateway

## Prerequisites 
- Kubernetes cluster
- kubectl 
- helm
- operator-sdk

## Install Kafka by OLM
```bash
# install olm
operator-sdk olm install

# install kafka operator
cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: Kafka.v1beta2.kafka.strimzi.io,KafkaBridge.v1beta2.kafka.strimzi.io,KafkaConnect.v1beta2.kafka.strimzi.io,KafkaConnectS2I.v1beta2.kafka.strimzi.io,KafkaConnector.v1beta2.kafka.strimzi.io,KafkaMirrorMaker.v1beta2.kafka.strimzi.io,KafkaMirrorMaker2.v1beta2.kafka.strimzi.io,KafkaRebalance.v1beta2.kafka.strimzi.io,KafkaTopic.v1beta2.kafka.strimzi.io,KafkaUser.v1beta2.kafka.strimzi.io
  name: kafka-operators
  namespace: kafka
spec:
  targetNamespaces:
    - kafka
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: strimzi-kafka-operator
  namespace: kafka
spec:
  channel: strimzi-0.32.x
  name: strimzi-kafka-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF

# kafka cluster
cat <<EOF | kubectl apply -f -                            
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-brokers-cluster
  namespace: kafka
spec:
  kafka:
    replicas: 3
    version: 3.3.1
    logging:
      type: inline
      loggers:
        kafka.root.logger.level: "INFO"
    readinessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
        configuration:
          useServiceDnsDomain: true
      - name: external
        port: 9095
        type: nodeport
        tls: true 
        # authentication: # the hostname of nodeport is not fixed, so we can't use tls, 
        #   type: tls     # https://strimzi.io/blog/2019/04/23/accessing-kafka-part-2/
        configuration:
          bootstrap:
            nodePort: 30095
    config:
      auto.create.topics.enable: "false"
      offsets.topic.replication.factor: 2
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      inter.broker.protocol.version: 3.3
      ssl.cipher.suites: "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
      ssl.enabled.protocols: "TLSv1.2"
      ssl.protocol: "TLSv1.2"
    storage:
      type: ephemeral
  zookeeper:
    replicas: 1
    logging:
      type: inline
      loggers:
        zookeeper.root.logger: "INFO"
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: event
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-brokers-cluster
spec:
  partitions: 1
  replicas: 2
  config:
    cleanup.policy: compact
EOF                                                                                                                                              <....
```

## Expose the Kafka Cluster By KafkaBridge

- Create the `KafkaBridge`

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: envoy-kafka-bridge
  namespace: kafka
spec:
  replicas: 1
  bootstrapServers: kafka-brokers-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
  http:
    port: 8080
EOF
```

- Verification

```bash
# forward 8080 by bridge pod
kubectl -n kafka port-forward $(kubectl get pods -l strimzi.io/cluster=envoy-kafka-bridge -n kafka -o jsonpath="{.items[0].metadata.name}") 8080:8080

# or forward 8080 by svc
kubectl -n kafka port-forward svc/$(kubectl get svc -l strimzi.io/cluster=envoy-kafka-bridge -n kafka -o jsonpath="{.items[0].metadata.name}") 8080:8080

# list topic
curl http://localhost:8080/topics

# send message to the topic
curl --location 'http://localhost:8080/topics/event' -H 'Content-Type: application/vnd.kafka.json.v2+json' --data \
'{
   "records":[
      {
         "key":"event1",
         "value":{ "hello":"world" }
      },
      {
         "key":"event2",
         "value":{ "foo":"doo" }
      }
   ]
}'

# register a kafka consumer in a new consumer group
curl -X POST http://localhost:8080/consumers/gloo-kafka-consumer-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "gloo-kafka-consumer",
    "auto.offset.reset": "earliest",
    "format": "json",
    "enable.auto.commit": false,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'

# subscribe to the topic
curl -X POST http://localhost:8080/consumers/gloo-kafka-consumer-group/instances/gloo-kafka-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "topics": [
        "event"
    ]
}'

# consume message with the consumer
while true; do curl -X GET http://localhost:8080/consumers/gloo-kafka-consumer-group/instances/gloo-kafka-consumer/records \
-H 'accept: application/vnd.kafka.json.v2+json'; sleep 1; done
```

## Install Envoy Gateway
```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -n envoy-gateway-system --create-namespace

# wait Envoy Gateway
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## Deploy an Envoy Sample with GatewayClass、Gateway and HTTPRoute on kafka namespace

- Proxy kafka with Envoy gateway

```bash
# get the kafkabridge service(8080)
kubectl get svc -n kafka envoy-kafka-bridge-bridge-service
KAFKA_SERVICE=$(kubectl get svc -l strimzi.io/cluster=envoy-kafka-bridge -n kafka -o jsonpath="{.items[0].metadata.name}")

# create the envoy proxy
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
  namespace: kafka
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-gateway
  namespace: kafka
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-route
  namespace: kafka
spec:
  parentRefs:
    - name: envoy-gateway
  hostnames:
    - "www.example.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: $KAFKA_SERVICE
          port: 8080
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
EOF
```

- Verification

```bash
# get the service of the gateway
kubectl get svc -n envoy-gateway-system --selector=gateway.envoyproxy.io/owning-gateway-namespace=kafka,gateway.envoyproxy.io/owning-gateway-name=envoy-gateway
export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system --selector=gateway.envoyproxy.io/owning-gateway-namespace=kafka,gateway.envoyproxy.io/owning-gateway-name=envoy-gateway -o jsonpath='{.items[0].metadata.name}')
echo $ENVOY_SERVICE

# access the application by local host
kubectl -n envoy-gateway-system port-forward service/${ENVOY_SERVICE} 8888:80
curl --verbose --header "Host: www.example.com" http://localhost:8888/topics

# access the application by LB
export GATEWAY_HOST=$(kubectl get svc/${ENVOY_SERVICE} -n envoy-gateway-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_HOST
curl --verbose --header "Host: www.example.com" http://$GATEWAY_HOST/topics
```

## Sending and Receiving Message by Envoy Gateway

```bash
# list topic
curl --header "Host: www.example.com" http://localhost:8888/topics

# send message to the topic
curl --header "Host: www.example.com" --location 'http://localhost:8888/topics/event' -H 'Content-Type: application/vnd.kafka.json.v2+json' --data \
'{
   "records":[
      {
         "key":"event3",
         "value":{ "hello":"world" }
      },
      {
         "key":"event4",
         "value":{ "foo":"doo" }
      }
   ]
}'

# register a kafka consumer in a new consumer group
curl --header "Host: www.example.com" -X POST http://localhost:8888/consumers/envoy-kafka-consumer-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "envoy-kafka-consumer",
    "auto.offset.reset": "earliest",
    "format": "json",
    "enable.auto.commit": true,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'

# subscribe to the topic
curl --header "Host: www.example.com" -X POST http://localhost:8888/consumers/envoy-kafka-consumer-group/instances/envoy-kafka-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "topics": [
        "event"
    ]
}'

# consume message with the consumer
while true; do curl --header "Host: www.example.com" -X GET http://localhost:8888/consumers/envoy-kafka-consumer-group/instances/envoy-kafka-consumer/records \
-H 'accept: application/vnd.kafka.json.v2+json'; sleep 1; done
```

## Uninstall Envoy
```bash
# Delete the GatewayClass、Gateway、HTTPRoute and other application resources
# kubectl delete -f https://github.com/envoyproxy/gateway/releases/download/latest/quickstart.yaml --ignore-not-found=true

# Delete the Envoy Gateway
helm uninstall eg -n envoy-gateway-system
```
