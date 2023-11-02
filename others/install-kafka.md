# Install the Strimzi Kafka by OLM
 
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