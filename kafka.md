## Introduction

Kafka is a distributed streaming platform originally conceived by LinkedIn but made open source in 2011. It is an incredibly powerful system to process and store streams of information, and due to its distributed nature is great for deploying on Kubernetes. This guide will walk you through deploying a Kafka cluster using [Banzai Cloud Kafka Operator](https://github.com/banzaicloud/kafka-operator). This guide is a condensed version of the official Banzai documentation, the idea being to quickly get up and running with Kafka.

By following this guide, you will provision a Kubernetes cluster, install the requisite software to run Kafka on it, deploy Kafka, and validate everything has gone to plan by employing a `producer` to serve a stream of information, and a `consumer` to receive it.

## Setup

_Prerequisites_:

If you do not yet have a Civo account enabled with access to the managed Kubernetes service, [register your interest here](https://www.civo.com/kube100). Once you are signed up, you will benefit from free credit for the duration of the beta. Other tools you will need are the following:

- **[Civo CLI](https://github.com/civo/cli)** (Not strictly necessary but it is _FAST_)
- **[Helm3](https://helm.sh/docs/intro/install/)**
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)**

### Provision Kubernetes

While you can use any kubernetes cluster to deploy Kafka according to this guide, I will use [Civo's](https://www.civo.com/) offering, meaning I can use the command-line tool to provision a cluster:

```bash
civo k8s create \
  --nodes 3 \
  --save --switch --wait \
  kafka
```

This will create a 3-node cluster called `kafka` and save the `KUBECONFIG` to our machine, switching the context to our new cluster so that you can easily get to work on it.

### Provision Dependencies

Let's get the core dependencies provisioned. We'll need:

- **Cert Manager**: cert manager off-loads the heavy lifting for certificate generation and rotation. There are many configurations that can be enabled, just check out the Banzai documentation for more information.
- **Prometheus Operator**: We only need the core bundle which will enable the use of [service monitors](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md#related-resources) and alerts. This is one of the key enablers that allows Banzai Kafka clusters to recover and rebalance data.
- **Zookeeper**: Zookeeper is the key value database which stores the Kafka state. You can use any Zookeeper endpoint but Banzai has packaged an operator for use. 

**Note**: This tutorial should be fully recreatable from the clodeblocks. However, it may be helpful to wait until each workload finishes deploying before moving on to the next command. I recommend putting a watch on the cluster pods in a separate terminal `watch kubectl get pods --all-namespaces`.

To keep things uniform, I have included the `kubectl` commands to install the various dependencies. You can also use the Civo Kubernetes marketplace to deploy Cert-Manager and Prometheus Operator directly onto your cluster. 

We'll also need the _Banzai Helm Repo_:
Make sure you have Helm 3 installed and run the following in order:

```bash
helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com/
```

_Cert-Manager_:

```bash
# create cert-manager deployment
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
```

_Prometheus Operator_:

```bash
# create the prometheus operator
kubectl apply -n default -f https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml
```

_Zookeeper Operator_:

```bash
# create zookeeper namespace
kubectl create namespace zookeeper

# create zk operator
helm upgrade --install \
  --namespace zookeeper \
  --wait \
  zookeeper-operator banzaicloud-stable/zookeeper-operator
```

_Zookeeper_:

```bash
# create zookeeper cluster
cat <<EOF | kubectl apply --namespace zookeeper -f -

apiVersion: zookeeper.pravega.io/v1beta1
kind: ZookeeperCluster
metadata:
  name: zookeepercluster
  namespace: zookeeper
spec:
  replicas: 3
EOF
```

### Provision Banzai Kafka

Again there are lots of configurations details this guide does not cover for reasons of brevity. Please see the [Banzai documentation](https://github.com/banzaicloud/kafka-operator) for more options.

Components:

- **Kafka Operator**: will maintain the lifecycle, data rebalancing and scaling for all the provisioned Kafkas in the cluster.
- **Kafka Instance**: this guide provisions a simple kafka instance, configured for internal cluster access and initialized with some basic scaling/rebalancing rules.

_Kafka Operator_:

```bash
# create the kafka namespace
kubectl create namespace kafka

# get the values file configured for prometheus
TMP_FILE=/tmp/kafka-prometheus-alerts.yaml

curl https://raw.githubusercontent.com/gabeduke/civo-kafka-example/v1.0.0/kafka-prometheus-alerts.yaml -o $TMP_FILE -s

# install kafka operator with prometheus alerts
helm upgrade --install \
  --namespace kafka \
  --values $TMP_FILE \
  kafka-operator banzaicloud-stable/kafka-operator
```

_Kafka_:

```bash
# create the kafka cluster
KAFKA_INSTANCE=https://raw.githubusercontent.com/gabeduke/civo-kafka-example/v1.0.0/kafka.yaml
curl $KAFKA_INSTANCE | kubectl apply -n kafka -f -

# create the service monitor

KAFKA_SERVICE_MONITOR=https://raw.githubusercontent.com/gabeduke/civo-kafka-example/v1.0.0/kafka-prometheus.yaml
curl $KAFKA_SERVICE_MONITOR | kubectl apply -n kafka -f -
```

### Validate

First we will validate the Cruise Control Dashboard is online and healthy. [Cruise Control](https://github.com/linkedin/cruise-control) is a tool from LinkedIn which provides exceptional operational control over Kafka clusters. The API can be triggered via Prometheus alerts, making Banzai clusters highly resilient. Take some time to ahead and explore the configuration options if you want to go deeper than the surface-level example here.

_Cruise Control Dashboard_:

```bash
# proxy to the cruise-control dashboard for kafka maintenance (may take a couple of minutes)
kubectl port-forward -n kafka svc/kafka-cruisecontrol-svc 8090:8090 &
echo http://localhost:8090
```

We can also validate that we can produce and consume from this cluster. First we need to provision a [topic](https://kafka.apache.org/intro) to use (_note_: the cluster must be finished provisioning before the topic can be applied):

```bash
# create a topic to which we can produce/consume
cat <<EOF | kubectl apply -f -

apiVersion: kafka.banzaicloud.io/v1alpha1
kind: KafkaTopic
metadata:
  name: civo-topic
  namespace: kafka
spec:
  clusterRef:
    name: kafka
  name: civo-topic
  partitions: 3
  replicationFactor: 2
  config:
    "retention.ms": "604800000"
    "cleanup.policy": "delete"
EOF
```

Run the following two commands in separate terminals:

_Produce_:

```bash
# run a producer in a pod
kubectl run kafka-producer \
  -n kafka -it --rm=true --restart=Never \
  --image=wurstmeister/kafka:2.12-2.3.0 \
    -- /opt/kafka/bin/kafka-console-producer.sh \
    --broker-list kafka-headless:29092 \
    --topic civo-topic
```

_Consume_:

```bash
# run a consumer in a pod
kubectl run kafka-consumer \
  -n kafka -it --rm=true --restart=Never \
  --image=wurstmeister/kafka:2.12-2.3.0 \
    -- /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka-headless:29092 \
    --from-beginning \
    --topic civo-topic
```

You will notice that anything you input into the "producer" terminal will be reflected in the "consume" terminal. This means Kafka is processing the stream of information, and is letting your consumer tap into the stream produced. Your Kafka cluster is working.

## Clean

Well that was fun. Go check out the [Banzai docs](https://github.com/banzaicloud/kafka-operator) and tune a cluster to your needs. Time to clean up!

```bash
# kill any dangling proxies
killall kubectl

# clean up the kubernetes cluster if you are done or want to retry the process
civo k8s delete kafka
```
