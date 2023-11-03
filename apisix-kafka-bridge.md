# Access the Kafka Cluster by APISIX Gateway

<!-- TOC -->

- [Access the Kafka Cluster by APISIX Gateway](#access-the-kafka-cluster-by-apisix-gateway)
  - [Prerequisites](#prerequisites)
  - [Install Strimzi Kafka Cluster](#install-strimzi-kafka-cluster)
  - [Expose the Kafka Cluster By KafkaBridge](#expose-the-kafka-cluster-by-kafkabridge)
  - [Running APISIX on Openshift](#running-apisix-on-openshift)
    - [Install APISIX on ROSA](#install-apisix-on-rosa)
    - [Optional: Install APISIX Dashboard](#optional-install-apisix-dashboard)
    - [Config the Kafka Route with Admin API](#config-the-kafka-route-with-admin-api)
    - [Request the Kafka Route with Client API](#request-the-kafka-route-with-client-api)
  - [Develop an Authentication Plugin with Golang](#develop-an-authentication-plugin-with-golang)
    - [Develop a Validate Cert Plugin](#develop-a-validate-cert-plugin)
    - [Build the APISIX Image with the above Plugin](#build-the-apisix-image-with-the-above-plugin)
    - [Startup the Plugin When Running the Server](#startup-the-plugin-when-running-the-server)
    - [Replace the Deployment Image](#replace-the-deployment-image)
    - [Verification](#verification)
    - [Reference](#reference)

<!-- /TOC -->

## Prerequisites

- Openshift cluster
- kubectl and oc
- operator-sdk
- helm

## [Install Strimzi Kafka Cluster](./../others/install-kafka.md)

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

### [Install APISIX on ROSA]((https://docs.api7.ai/apisix/install/kubernetes/rosa))

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

### Optional: Install APISIX Dashboard

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

### Config the Kafka Route with Admin API

```bash
# get the kafkabridge service(8080)
KAFKA_NAMESPACE=kafka
KAFKA_SERVICE=$(kubectl get svc -l strimzi.io/cluster=strimzi-kafka-bridge -n $KAFKA_NAMESPACE -o jsonpath="{.items[0].metadata.name}")

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

## Develop an Authentication Plugin with Golang

### [Develop a Validate Cert Plugin](https://github.com/apache/apisix-go-plugin-runner/commit/84adcb2447287d48419c312f8aba8039c4b1f32d#diff-940fa3214bf2b0eb733540bb2e25fa7d6fbfea4821aa73da98911914a07a54db)

### Build the APISIX Image with the above Plugin

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

### Startup the Plugin When Running the Server

Note: Modify the `config.yaml` by `apisix` ConfigMap

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

### Replace the Deployment Image

```bash
# image: quay.io/myan/apisix-360-go:0.1
kubectl set image deployment/apisix apisix=quay.io/myan/apisix-360-go:0.1
```

### [Verification](./rest/apisix-client.rest)

### Reference

  - https://apisix.apache.org/zh/docs/ingress-controller/tutorials/how-to-use-go-plugin-runner-in-apisix-ingress/
  - https://www.apiseven.com/blog/go-makes-apache-apisix-better
  - https://zhuanlan.zhihu.com/p/613540331
  - https://www.fdevops.com/2022/10/09/casbin-apisix-31182
  - https://blog.csdn.net/weixin_42873928/article/details/123279381