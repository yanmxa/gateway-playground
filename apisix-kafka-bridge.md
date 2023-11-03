# Access the Kafka Cluster by APISIX Gateway

## Prerequisites

- Openshift cluster
- kubectl 
- helm
- operator-sdk

## [Install Kafka](./../others/install-kafka.md)

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
# kill -9 $(lsof -t -i:9080)
kubectl -n ${KAFKA_NAMESPACE} port-forward $(kubectl get pods -l strimzi.io/cluster=strimzi-kafka-bridge -n ${KAFKA_NAMESPACE} -o jsonpath="{.items[0].metadata.name}") 8080:8080

# or forward 8080 by svc
kubectl -n ${KAFKA_NAMESPACE} port-forward svc/$(kubectl get svc -l strimzi.io/cluster=strimzi-kafka-bridge -n ${KAFKA_NAMESPACE} -o jsonpath="{.items[0].metadata.name}") 8080:8080

# list topic
curl http://localhost:8080/topics

# consume message with the consumer
while true; do curl -X GET http://localhost:8080/consumers/strimzi-kafka-consumer-group/instances/strimzi-kafka-consumer/records \
-H 'accept: application/vnd.kafka.json.v2+json'; sleep 1; done
```

## Running APISIX on Openshift

### Develope a Customize Plugin

- Build the image with plugin

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

- Reference

  - https://apisix.apache.org/zh/docs/ingress-controller/tutorials/how-to-use-go-plugin-runner-in-apisix-ingress/
  - https://www.apiseven.com/blog/go-makes-apache-apisix-better
  - https://zhuanlan.zhihu.com/p/613540331
  - https://www.fdevops.com/2022/10/09/casbin-apisix-31182
  - https://blog.csdn.net/weixin_42873928/article/details/123279381


### Install APISIX with the Built Image

- [Install APISIX on ROSA]((https://docs.api7.ai/apisix/install/kubernetes/rosa))

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

- Optional: Install APISIX Dashboard

```bash
helm install apisix-dashboard apisix/apisix-dashboard --namespace apisix

# expose the dashboard
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: apisix-dashboard-lb
  namespace: apisix
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app.kubernetes.io/instance: apisix-dashboard
    app.kubernetes.io/name: apisix-dashboard
  type: LoadBalancer
EOF
```

- Startup the plugin when running the server(`config.yaml`)

```yaml
  etcd:
    host:                          # it's possible to define multiple etcd hosts addresses of the same etcd cluster.
      - "http://apisix-etcd.apisix.svc.cluster.local:2379"
    prefix: "/apisix"    # configuration prefix in etcd
    timeout: 30    # 30 seconds
...
# Nginx will hide all environment variables by default. So you need to declare your variable first in the conf/config.yaml
# https://github.com/apache/apisix/blob/master/docs/en/latest/external-plugin.md
nginx_config:
  envs:
    - APISIX_LISTEN_ADDRESS
    - APISIX_CONF_EXPIRE_TIME

ext-plugin:
  # path_for_test: "/tmp/runner.sock" 
  cmd: ["/usr/local/apisix/apisix-go-plugin-runner/go-runner", "run", "-m", "prod"]
```

- Replace the image of the deployment

```bash
# image: quay.io/myan/apisix-360-go:0.1
kubectl set image deployment/apisix apisix=quay.io/myan/apisix-360-go:0.1
```

### Config the Kafka Route with Admin API

- Forward Admin Server to Local Host

```bash
# get the kafkabridge service(8080)
KAFKA_NAMESPACE=kafka
kubectl get svc -n $KAFKA_NAMESPACE strimzi-kafka-bridge-bridge-service
KAFKA_SERVICE=$(kubectl get svc -l strimzi.io/cluster=strimzi-kafka-bridge -n $KAFKA_NAMESPACE -o jsonpath="{.items[0].metadata.name}")
# KAFKA_SERVICE=strimzi-kafka-bridge-bridge-service

# configure the api by apisix pod
# kubectl exec -it $(kubectl get pods -l "app.kubernetes.io/name=apisix,app.kubernetes.io/instance=apisix" -n apisix -o jsonpath="{.items[0].metadata.name}") -n apisix -c apisix -- sh

# forward 9180 port to local host
kubectl -n apisix port-forward $(kubectl get pods -l app.kubernetes.io/name=apisix -n apisix -o jsonpath="{.items[0].metadata.name}") 9180:9180

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
  "plugins": {
    "ext-plugin-post-resp": {
      "conf": [
        {"name":"my-response-rewrite", "value":"{\"tag\":\"\"}"}
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

# test
curl -i -H "Host: example.com" -X GET "http://127.0.0.1:9080/topics"
```

### Request the Kafka Route with Client API

```bash
# forward the http api of apisix to local host
kubectl -n apisix port-forward $(kubectl get pods -l app.kubernetes.io/name=apisix -n apisix -o jsonpath="{.items[0].metadata.name}") 9080:9080

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

## Develop an Authentication Plugin Using Payload
