# Autoscaling a Dapr app with KEDA

Dapr is a programming model that's being installed and operated using a sidecar, and thus leaves autoscaling to the hosting layer, for example Kubernetes.
Many of Dapr's [bindings](../../concepts/bindings#supported-bindings-and-specs) overlap with those of [KEDA](https://github.com/kedacore/keda), an Event Driven Autoscaler for Kubernetes.

For apps that use these bindings, its easy to configure a KEDA autoscaler.

## Install KEDA

To install KEDA, follow [these instructions](https://github.com/dapr/docs/tree/master/concepts/bindings#supported-bindings-and-specs) on the KEDA Github page.

## Create KEDA enabled Dapr binding

For this example, we'll be using Kafka.<br>
You can install Kafka in your cluster by using Helm:

```
$ helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
$ helm install --name my-kafka incubator/kafka
```

Next, we'll create the Dapr Kafka binding for Kubernetes.<br>
Paste the following in a file named kafka.yaml:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafkaevent
spec:
  type: bindings.kafka
  metadata:
  - name: brokers
    value: "my-kafka:9092"
  - name: topics
    value: "myTopic"
  - name: consumerGroup
    value: "group1"
```

The following YAML defines a Kafka component that listens for the topic `myTopic`, with consumer group `group1` and that connects to a broker at `my-kafka:9092`.

Deploy the binding to the cluster:

```
$ kubectl apply -f kafka.yaml
```

## Create the KEDA autoscaler for Kafka

Paste the following to a file named kafka_scaler.yaml, and put the name of your Deployment in the required places:

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaler
  namespace: default
  labels:
    deploymentName: <REPLACE-WITH-DEPLOYMENT-NAME>
spec:
  scaleTargetRef:
    deploymentName: <REPLACE-WITH-DEPLOYMENT-NAME>
  triggers:
  - type: kafka
    metadata:
      type: kafkaTrigger
      direction: in
      name: event
      topic: myTopic
      brokers:  my-kafka:9092
      consumerGroup: group2
      dataType: binary
      lagThreshold: '5'
```

Deploy the KEDA scaler to Kubernetes:

```
$ kubectl apply -f kafka_scaler.yaml
```

All done!

You can now start publishing messages to your Kafka topic `myTopic` and watch the pods autoscale when the lag threshold is bigger than `5`, as defined in the KEDA scaler manifest.
