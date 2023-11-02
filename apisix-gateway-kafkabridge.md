# Access the Kafka Cluster by Envoy Gateway

## Prerequisites 
- Openshift cluster
- kubectl 
- helm
- operator-sdk

## [Install Kafka](./others/install-kafka.md)

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

## Install APISix on Openshift

### Develope a Customize Plugin and rebuild the APISix Image

- Develop the plugin with golang

```bash
git clone git@github.com:apache/apisix-go-plugin-runner.git
...
# build binary
make build
# create Dockerfile to add the build binary
`Dockerfile
FROM apache/apisix:3.6.0-debian
COPY ./go-runner /usr/local/apisix/apisix-go-plugin-runner/go-runner
`
# build and push image
docker build -f ./Dockerfile -t quay.io/myan/apisix-360-go:0.1 .
docker push quay.io/myan/apisix-360-go:0.1
```

- Edit the `config.yaml` to add plugin startup command

```yaml
ext-plugin:
  cmd: ["/usr/local/apisix/apisix-go-plugin-runner/go-runner", "run"]
```

- Reference

  - https://apisix.apache.org/zh/docs/ingress-controller/tutorials/how-to-use-go-plugin-runner-in-apisix-ingress/
  - https://www.apiseven.com/blog/go-makes-apache-apisix-better
  - https://zhuanlan.zhihu.com/p/613540331
  - https://www.fdevops.com/2022/10/09/casbin-apisix-31182
  - https://blog.csdn.net/weixin_42873928/article/details/123279381


### [Install APISIX on ROSA](https://docs.api7.ai/apisix/install/kubernetes/rosa)

```bash
oc create sa apisix-sa -n apisix
oc adm policy add-scc-to-user anyuid -z apisix-sa -n apisix

helm install apisix apisix/apisix \
  --set gateway.type=NodePort \
  --set etcd.podSecurityContext.enabled=false \
  --set etcd.containerSecurityContext.enabled=false \
  --set serviceAccount.name=apisix-sa \
  --namespace apisix
```

- Startup the plugin when running the server

```yaml
# edit the apisix configmap to start the plugin
ext-plugin:
  cmd: ["/usr/local/apisix/apisix-go-plugin-runner/go-runner", "run"]
```

- Replace the image of the deployment

```yaml
# image: quay.io/myan/apisix-360-go:0.1
kubectl set image deployment/apisix apisix=quay.io/myan/apisix-360-go:0.1
```

// TODO
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
