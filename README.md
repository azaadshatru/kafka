# Deploy Kafka on Open Source Kubernetes
#### Apache Kafka is a distributed event store and stream-processing platform. It is an open-source system developed by the Apache Software Foundation written in Java and Scala. The project aims to provide a unified, high-throughput, low-latency platform for handling real-time data feeds.

### Defining a Kafka Namespace
#### First, we define a namespace for deploying all Kafka resources, using a file named 01-namespace.yaml:

```
apiVersion: v1
kind: Namespace
metadata:
  name: "kafka"
  labels:
    name: "kafka"
```

#### We apply this file using kubectl apply -f 01-namespace.yaml.

#### We can test that the namespace was created correctly by running kubectl get namespaces, verifying that Kafka is a namespace present in Minikube.

### Deploying Zookeeper
#### Next, we deploy Zookeeper to our k8s namespace. We create a file name 02-zookeeper.yaml with the following contents:
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: zookeeper-service
  name: zookeeper-service
  namespace: kafka
spec:
  type: NodePort
  ports:
    - name: zookeeper-port
      port: 2181
      nodePort: 30181
      targetPort: 2181
  selector:
    app: zookeeper
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: zookeeper
  name: zookeeper
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - image: wurstmeister/zookeeper
          imagePullPolicy: IfNotPresent
          name: zookeeper
          ports:
            - containerPort: 2181
```
#### There are two resources created in this YAML file. The first is the service called zookeeper-service, which will use the deployment created in the second resource named zookeeper. The deployment uses the wurstmeister/zookeeper Docker image for the actual Zookeeper binary. The service exposes that deployment on a port on the internal k8s network. In this case, we use the standard Zookeeper port of 2181, which the Docker container also exposes.

#### We apply this file with the following command: 

```
$kubectl apply -f 02-zookeeper.yaml

```
#### We can test for the successfully created service as follows:
```
$ kubectl get services -n kafka
NAME               TYPE      CLUSTER-IP     PORT(S)         AGE
zookeeper-service  NodePort  10.100.69.243  2181:30181/TCP  3m4s
```
#### We see the internal IP address of Zookeeper (10.100.69.243), which we’ll need to tell the broker where to listen for it.

### Deploying a Kafka Broker

#### The last step is to deploy a Kafka broker. We create a 03-kafka.yaml file with the following contents, be we replace ZOOKEEPER-INTERNAL-IP with the CLUSTER-IP from the previous step for Zookeeper. The broker will fail to deploy if this step is not taken.

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kafka-broker
  name: kafka-service
  namespace: kafka
spec:
  type: NodePort
  ports:
    - name: kafka-port
      port: 9092
      nodePort: 9092
      targetPort: 9092
  selector:
    app: kafka-broker
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kafka-broker
  name: kafka-broker
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-broker
  template:
    metadata:
      labels:
        app: kafka-broker
    spec:
      hostname: kafka-broker
      containers:
      - env:
        - name: KAFKA_BROKER_ID
          value: "1"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: <ZOOKEEPER-INTERNAL-IP>:2181
        - name: KAFKA_LISTENERS
          value: PLAINTEXT://:9092
        - name: KAFKA_ADVERTISED_LISTENERS
          value: PLAINTEXT://<KAFKA HOST IP OR LOCALHOST OR 127.0.0.1>:9092
        image: wurstmeister/kafka
        imagePullPolicy: IfNotPresent
        name: kafka-broker
        ports:
        - containerPort: 9092

```
#### Again, we are creating two resources — service and deployment — for a single Kafka Broker. We run kubectl apply -f 03-kafka.yaml. We verify this by seeing the pods in our namespace:

```
$ kubectl get pods -n kafka
NAME                            READY   STATUS    RESTARTS   AGE
kafka-broker-5c55f544d4-hrgnv   1/1     Running   0          48s
zookeeper-55b668879d-xc8vd      1/1     Running   0          35m
```

#### The Kafka Broker pod might take a minute to move from ContainerCreating status to Running status.
#### Notice the line in 03-kafka.yaml where we provide a value for KAFKA_ADVERTISED_LISTENERS. To ensure that Zookeeper and Kafka can communicate by using this hostname (kafka-broker), we need to add the following entry to the /etc/hosts file on our local machine:
```
127.0.0.1 kafka-broker
```

#### I Prefer to use IP Address instead of kafka-broker hostname
