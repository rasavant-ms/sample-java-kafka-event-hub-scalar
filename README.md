# Sample for Kafka tigger scaler with KEDA using Event Hubs Kafka

Quickstart for Kafka trigger scaler using Azure Event Hubs Kafka head to scale Java applications deployed on AKS which consumes the event hub messages.

# Deploy AKS cluster

The Azure portal was used to deploy basic AKS service using instructions in the [Azure docs](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster).

```bash
# Create AKS cluster
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --generate-ssh-keys

# Install Kubernetes CLI
az aks install-cli

# Connect to cluster using kubectl
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify connection to cluster
$ kubectl get nodes

NAME                       STATUS   ROLES   AGE   VERSION
aks-nodepool1-12345678-0   Ready    agent   32m   v1.13.10
```

The resulting ARM template of this deployment is available as [AKS-deployment-KEDA-ARMtemplate.zip](./AKS-deployment-KEDA-ARMtemplate.zip) for download.

# Deploy KEDA on the AKS cluster

Detailed instructions for this are available [here](https://keda.sh/deploy/).

```bash
# Add KEDA Helm repo
helm repo add kedacore https://kedacore.github.io/charts

# Update KEDA Helm repo
helm repo update

# Install KEDA Helm chart for Helm 3
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

# Verify KEDA operator was deployed correctly
kubectl -n keda get pods

NAME                            READY     STATUS    RESTARTS   AGE
keda-operator-576c5dfcdf-bjmtd   2/2       Running   0          42s
```

# Create Event Hub to act as Scalar

Documentation for [creating Event Hub can be found here](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-quickstart-cli). Make sure that Standard Tier is selected and Kafka is enabled.

```bash
# Create EventHub namespace
az eventhubs namespace create --name myEventHubNamespace \
                              --resource-group myResourceGroup \
                              --capacity 10 \
                              --enable-auto-inflate true \
                              --enable-kafka true \
                              --location EastUS \
                              --maximum-throughput-units 20 \
                              --sku Standard

# Create EventHub
az eventhubs eventhub create --name myEventHub \ 
                             --namespace-name myEventHubNamespace \
                             --resource-group myResourceGroup \
                             --partition-count 10

# Create ConsumerGroup
az eventhubs eventhub consumer-group create --eventhub-name myEventHub\
                                            --name myConsumerGroup \
                                            --namespace-name myEventHubNamespace\
                                            --resource-group myResourceGroup
```

# Store secrets in `deploy-kafka-secret.yaml`

## Get EventHub ConnectionString

This can be obtained from the Azure portal or via commandline as shown below

```
az eventhubs namespace authorization-rule keys list --resource-group myResourceGroup --namespace-name myEventHubNamespace --name RootManageSharedAccessKey
```
Copy value of `primaryConnectionString`

## Base64 encoding of secrets

All the secrets need to be base64 encoded as shown

```bash
echo -n "<eventhub_connectionstring>" | base64
```

Enter the base64 encoded Event Hub Connection String secret in the `deploy-secret.yaml` file against the `password` field.

Enter the other base64 encoded values against the corresponding fields in the `deply-secret.yaml` file.

```yaml
password: "<base64_encoded_eventhub_conmectionstring>"
eventhub_namespace: "<base64_encoded_eventhub_namespace>"
consumer_group: "<base64_encoded_consumer_group_name>"
eventhub_name: "<base64_encoded_eventhub_name>"
```

The other secrets in the `deploy-secret.yaml` are already filled in for you. The `authmode` is set to be base64 encoded string for `sasl_ssl_plain` and the `username` is set to be base64 encoded string for `$ConnectionString`.

```yaml
authMode: "c2FzbF9zc2xfcGxhaW4="
username: "JENvbm5lY3Rpb25TdHJpbmc="
```

# Build and Dockerize JAVA application

The Java application can be built independently with running below command from the root directory of this repo

```bash
mvn clean package
```

Verify that the `Dockerfile` has the right path for target

```bash
# Create docker image for Java producer application
docker build -t <tag_name> .
```

Create ACR to store docker image

```bash
# Create ACR
az acr create --resource-group MyResourceGroup --name myacr --sku Basic

# Merge AKS cluster to current context
az aks get-credentials --name myAKSCluster --resource-group MyResourceGroup

# Integrate ACR to existing AKS cluster
az aks update -n myAKSCluster -g MyResourceGroup --attach-acr myacr
```

Upload docker image to ACR

```bash
# Login to ACR
az acr login --name myacr

# Get ACR server name
az acr list --resource-group MyResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
>AcrLoginServer
------------------
myacr.azurecr.io

# Tag docker image to created ACR
docker tag <image_tag> myacr.azurecr.io/<image_tag>

# Publish docker image to created ACR
docker push myacr.azurecr.io/<image_tag>
```

# Update `deploy-kafka-scalar.yaml`

## Update deployment image details

Update the Deployment container spec with the image name obtained from previous step in the `deploy-kafka-scalar.yaml` file.

```yaml
    spec:
      containers:
      - name: kedaconsumer
        image: myacr.azurecr.io/<image_tag>
        env:
```

## Update ScaledObject metadata

The Kafka trigger needs some metadata to make sure it is scaling the pods based on the right details. These details are provided as a part of metadata in the `ScaledObject` deployment yaml.

Open the `deploy-kafka-scalar.yaml` file and replace the `<EVENTHUB_NAMESPACE>`, `<CONSUMER_GROUP>` and `<EVENTHUB_NAME>` with the actual values in plain text:

```yaml
    metadata:
      # Required
      brokerList: <EVENTHUB_NAMESPACE>.servicebus.windows.net:9093
      consumerGroup: <CONSUMER_GROUP>       # Make sure that this consumer group name is the same one as the one that is consuming topics
      topic: <EVENTHUB_NAME>
      lagThreshold: "50"
```

# Deploy secrets and application to AKS

```bash
# Deploy secrets
kubectl apply -f deploy-secret.yaml
> secret/keda-kafka-secrets created
```

Make sure that the image name and tag in the `deploy-kafka-scalar.yaml` file matches the docker image pushed to ACR in the previous section

```bash
# Deploy Java application as Deployment and ScaledObject details
kubectl apply -f deploy-kafka-scalar.yaml
> deployment.apps/kedaconsumer created
> triggerauthentication.keda.k8s.io/keda-trigger-auth-kafka-credential created
> scaledobject.keda.k8s.io/kafka-scaledobject created
```

Verify deployment

```bash
# Get pods -n keda
kubectl get pods -n keda
NAME                             READY   STATUS    RESTARTS   AGE
keda-operator-657c678b64-6p8hw   2/2     Running   0          26h

# Get deployment
# Since there are no messages in the Event Hub, the kedaconsumer has no pods allocated
kubectl get deployment
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
kedaconsumer    0/0     0            0           109m

# Get deployment -n keda
kubectl get deployment -n keda
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
keda-operator   1/1     1            1           25h

# Verify HPA is deployed
kubectl get hpa
NAME                    REFERENCE                 TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-kedaconsumer   Deployment/kedaconsumer   <unknown>/50 (avg)   1         100       0          117m
```

# Sending messages to Event Hub to test pod autoscaling

For this, you can execute your own producer sending a messages so that they exceed the lag threshold that was set in the `deply-kafka-scalar.yaml` file in order to see the pods scale up.

The producer sample given in the `producer` folder can also be used to send messages.

## Using given producer sample

### Prerequisites

* [Java Development Kit (JDK) 1.7+](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
    * On Ubuntu, run `apt-get install default-jdk` to install the JDK.
    * Be sure to set the JAVA_HOME environment variable to point to the folder where the JDK is installed.
* [Download](http://maven.apache.org/download.cgi) and [install](http://maven.apache.org/install.html) a Maven binary archive
    * On Ubuntu, you can run `apt-get install maven` to install Maven.

### Update configs

Modify the `producer/src/main/resources/producer.config` file with the Event Hub namespace and Event Hub ConnectionString values:

```config
bootstrap.servers=<EVENTHUB_NAMESPACE>.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="$ConnectionString" password="<EVENTHUB_CONNECTIONSTRING>";
```

Also make sure that the `<EVENTHUB_NAME>` is replaced with the right value in the `producer/src/main/java/TestProducer.java` file.

### Run the producer

To run the producer from the command line, generate the JAR and then run from within Maven (alternatively, generate the JAR using Maven, then run in Java by adding the necessary Kafka JAR(s) to the classpath):

```bash
# Run these commands from the producer directory
mvn clean package
mvn exec:java -Dexec.mainClass="TestProducer"
```

The producer will now begin sending events to the Kafka-enabled Event Hub at topic you chose and printing the events to stdout.

## Expected outcome

Once you let the producer run for a couple of seconds, you should start seeing the HPA scale out the number of pods. Ideally, depending on the number of messages being sent,  the number of pods should not be greater than the number of partitions on the Event Hub.

```bash
# Get HPA 
kubectl get hpa -w
NAME                    REFERENCE                 TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-kedaconsumer   Deployment/kedaconsumer   <unknown>/50 (avg)   1         100       0          30m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   <unknown>/50 (avg)   1         100       1          30m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   236/50 (avg)         1         100       1          31m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   50250m/50 (avg)      1         100       4          31m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   40/50 (avg)          1         100       5          31m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   25200m/50 (avg)      1         100       5          33m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   42800m/50 (avg)      1         100       5          33m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   40/50 (avg)          1         100       5          33m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   0/50 (avg)           1         100       5          34m
keda-hpa-kedaconsumer   Deployment/kedaconsumer   <unknown>/50 (avg)   1         100       0          38m

# Get pods
kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
kedaconsumer-57f88dc9c8-6stzx    1/1     Running   0          8s
kedaconsumer-57f88dc9c8-9fs5q    1/1     Running   0          7s
kedaconsumer-57f88dc9c8-jkt6f    1/1     Running   0          7s
kedaconsumer-57f88dc9c8-vtr5m    1/1     Running   0          7s
kedaconsumer-57f88dc9c8-hdw78    1/1     Running   0          7s
```