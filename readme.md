
# Kafka and MySQL Integration on Kubernetes

## Step 1: Download and Build Debezium MySQL Connector

```bash
sudo curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.0.0.Final/debezium-connector-mysql-1.0.0.Final-plugin.tar.gz | tar xvz
```

```Dockerfile
FROM strimzi/kafka:0.20.1-kafka-2.5.0
USER root:root
RUN mkdir -p /opt/kafka/plugins/debezium
COPY ./debezium-connector-mysql/ /opt/kafka/plugins/debezium/
USER 1001
```

```bash
docker build . -t henriksin1/connect-debezium
docker push henriksin1/connect-debezium
```

## Step 2: Create Kubernetes Namespace

```bash
kubectl apply -f 0-namespace.yaml
```

## Step 3: Deploy Strimzi Operators

Install Strimzi operators using either OLM or Helm:

### Using OLM

```bash
curl -L https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh -o install.sh
chmod +x install.sh
./install.sh v0.26.0
```

### Using Helm

```bash
helm repo add strimzi https://strimzi.io/charts
helm install my-strimzi-kafka-operator strimzi/strimzi-kafka-operator --version 0.38.0 -n kafka
```

```bash
kubectl create -f https://operatorhub.io/install/strimzi-kafka-operator.yaml
```

### Operators Included:

- **Cluster Operator:** Manages Kafka clusters, automating setup of Kafka brokers.
- **Topic Operator:** Manages creation, configuration, and deletion of topics.
- **User Operator:** Manages Kafka users requiring access to Kafka brokers.

## Step 4: Deploy Secrets

```bash
kubectl -n kafka create secret generic debezium-secret --from-file=secrets.properties
```

Create the file `secrets.properties`.

## Step 5: Deploy RBAC Access Control

```bash
kubectl apply -f rbac-debezium-role.yaml
```

## Step 6: Deploy Service Account

```bash
kubectl apply -f serviceaccount.yaml
```

## Step 7: Deploy RBAC Cluster Binding

```bash
kubectl apply -f rbac-cluster-binding.yaml
```

## Step 8: Deploy Kafka Cluster

```bash
kubectl apply -f kafka-kluster.yaml
```

Confirm the cluster is running:

```bash
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```

Exec into the Kafka cluster to run topics:

```bash
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
```

Open another terminal to confirm the topics are consumed:

```bash
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```

## Step 9: Deploy MySQL Cluster Database

```bash
kubectl apply -f 6-sql.yaml
```

Confirm the database is running:

```bash
kubectl describe service mysql -n kafka
```

Connect with your local MySQL Workbench.

## Step 10: Run Kafka Connector

Deploy Kafka Connect:

```bash
kubectl apply -f kafka-connect.yaml
```

Confirm all pods are running:

```bash
kubectl get all -n kafka
```

Run:

```bash
kubectl logs debezium-connect-cluster-connect-0 -n kafka
```

## Step 11: Create Kafka Connector File

```bash
kubectl apply -f debezium-connector.yaml
```

Run:

```bash
kubectl describe kafkaconnector debezium-connector-mysql -n kafka
```

Now it's showtime:

```bash
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

```bash
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mysql.inventory.customers
```

Exec into the database:

```bash
kubectl exec -it mysql-6597659cb8-j9rk4 -n kafka -- sh
```

```bash
mysql -h mysql -u root -p
```

Password:

```bash
use inventory;
show tables;
```

```bash
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mysql --from-beginning
```

Pictures and documentation on connecting to the SQL database serving as a microservice.
```

Feel free to adjust and enhance the documentation as needed.