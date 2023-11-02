# Access the Kafka Cluster by Envoy Gateway

## Prerequisites 
- Openshift cluster
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
# namespace
KAFKA_NAMESPACE=kafka

# create kafkabridge
cat <<EOF | oc apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: strimzi-kafka-bridge
  namespace: ${KAFKA_NAMESPACE}
spec:
  bootstrapServers: kafka-kafka-bootstrap.${KAFKA_NAMESPACE}.svc:9092
  http:
    port: 8080
  replicas: 1
EOF
```

- Verification

```bash
KAFKA_NAMESPACE=kafka

# forward 8080 by bridge pod
kubectl -n ${KAFKA_NAMESPACE} port-forward $(kubectl get pods -l strimzi.io/cluster=strimzi-kafka-bridge -n ${KAFKA_NAMESPACE} -o jsonpath="{.items[0].metadata.name}") 8080:8080

# or forward 8080 by svc
kubectl -n ${KAFKA_NAMESPACE} port-forward svc/$(kubectl get svc -l strimzi.io/cluster=strimzi-kafka-bridge -n ${KAFKA_NAMESPACE} -o jsonpath="{.items[0].metadata.name}") 8080:8080

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
curl -X POST http://localhost:8080/consumers/strimzi-kafka-consumer-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "strimzi-kafka-consumer",
    "auto.offset.reset": "earliest",
    "format": "json",
    "enable.auto.commit": false,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'

# subscribe to the topic
curl -X POST http://localhost:8080/consumers/strimzi-kafka-consumer-group/instances/strimzi-kafka-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "topics": [
        "event"
    ]
}'

# consume message with the consumer
while true; do curl -X GET http://localhost:8080/consumers/strimzi-kafka-consumer-group/instances/strimzi-kafka-consumer/records \
-H 'accept: application/vnd.kafka.json.v2+json'; sleep 1; done
```

## Install Apisix on Openshift

### Develope a plugin for the apisix

- https://apisix.apache.org/zh/docs/ingress-controller/tutorials/how-to-use-go-plugin-runner-in-apisix-ingress/
- https://www.apiseven.com/blog/go-makes-apache-apisix-better
- https://zhuanlan.zhihu.com/p/613540331
- others
```bash
make build

# create Dockerfile with following content
FROM apache/apisix:3.6.0-debian

COPY ./go-runner /usr/local/apisix-go-plugin-runner/go-runner

# build and push image
docker build -f ./Dockerfile -t quay.io/myan/apisix-360-go:0.1 .
docker push quay.io/myan/apisix-360-go:0.1
```




### Referrence
- https://docs.api7.ai/apisix/install/kubernetes/rosa
- https://apisix.apache.org/zh/docs/helm-chart/apisix/
- https://apisix.apache.org/zh/docs/helm-chart/apisix-dashboard/

## Deploy an Envoy Sample with GatewayClass、Gateway and HTTPRoute on kafka namespace

- Proxy kafka with APISix Gateway

```bash
# get the kafkabridge service(8080)
KAFKA_NAMESPACE=kafka
kubectl get svc -n $KAFKA_NAMESPACE strimzi-kafka-bridge-bridge-service
KAFKA_SERVICE=$(kubectl get svc -l strimzi.io/cluster=strimzi-kafka-bridge -n $KAFKA_NAMESPACE -o jsonpath="{.items[0].metadata.name}")

# configure the api by apisix pod
kubectl exec -it $(kubectl get pods -l "app.kubernetes.io/name=apisix,app.kubernetes.io/instance=apisix" -n apisix -o jsonpath="{.items[0].metadata.name}") -n apisix -c apisix -- sh

# create an upstream(upstream_id = 1)
curl "http://127.0.0.1:9180/apisix/admin/upstreams/1" \
-H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "type": "roundrobin",
  "nodes": {
    "strimzi-kafka-bridge-bridge-service.multicluster-global-hub.svc:8080": 1
  }
}'

# create a route
curl "http://127.0.0.1:9180/apisix/admin/routes/1" \
-H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "methods": ["GET"],
  "host": "example.com",
  "uri": "/*",
  "upstream_id": "1"
}'

# or
curl "http://127.0.0.1:9180/apisix/admin/routes/1" \
-H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "methods": ["GET", "POST", "DELETE", "PUT"],
  "host": "example.com",
  "uri": "/*",
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "strimzi-kafka-bridge-bridge-service.multicluster-global-hub.svc:8080": 1
    }
  }
}'

# test
curl -i -H "Host: example.com" -X GET "http://127.0.0.1:9080/topics"
```

- Verification

```bash
# forward the http api of apisix to local host
kubectl -n apisix port-forward $(kubectl get pods -l app.kubernetes.io/instance -n apisix -o jsonpath="{.items[0].metadata.name}") 9080:9080

# list topic
curl --verbose --header "Host: example.com" http://localhost:9080/topics

# send message to the topic
curl --header "Host: example.com" --location 'http://localhost:9080/topics/event' -H 'Content-Type: application/vnd.kafka.json.v2+json' --data \
'{
   "records":[
      {
         "key":"event5",
         "value": "hello5"
      },
      {
         "key":"event6",
         "value": "world6"
      }
   ]
}'

# create a kafka consumer in a new consumer group
curl --header "Host: example.com" -X POST http://localhost:9080/consumers/strimzi-kafka-consumer-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "strimzi-kafka-consumer",
    "auto.offset.reset": "earliest",
    "format": "json",
    "enable.auto.commit": true,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'

# subscribe to the topic
curl --header "Host: example.com" -X POST http://localhost:9080/consumers/strimzi-kafka-consumer-group/instances/strimzi-kafka-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "topics": [
        "event"
    ]
}'

# consume message with the consumer
while true; do curl --header "Host: example.com" -X GET http://localhost:9080/consumers/strimzi-kafka-consumer-group/instances/strimzi-kafka-consumer/records \
-H 'accept: application/vnd.kafka.json.v2+json'; sleep 1; done
```

## Start authentication with go plugin of apisix



### Add the plugin runner image to config.yaml

```bash
# edit cm apisix to add image: quay.io/myan/apisix-310-go:0.1
apisix:
  image: quay.io/myan/apisix-310-go:0.1

# then check the apisix server config: /usr/local/apisix/conf/config.yaml

curl "http://127.0.0.1:9180/apisix/admin/routes/1" \
-H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "methods": ["GET", "POST", "DELETE", "PUT"],
  "host": "example.com",
  "uri": "/*",
  "plugins": {
        "ext-plugin-post-resp": {
            "conf" : [
                {"name": "my-response-rewrite", "value": "{\"tag\":\"hello my response\"}"}
            ]
        }
  },
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "strimzi-kafka-bridge-bridge-service.multicluster-global-hub.svc:8080": 1
    }
  }
}'
```
