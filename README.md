
# Kafka as Kubernetes StatefulSet

Example of three Kafka brokers depending on three Zookeeper instances base on [GCP](https://console.cloud.google.com/kubernetes) .

To get consistent service DNS names `kafka-N.broker.kafka`(`.svc.cluster.local`), run everything in a [namespace](http://kubernetes.io/docs/admin/namespaces/walkthrough/):
```
kubectl create -f namespace.yml
```

## Set up volume claims

You may add [storage class](http://kubernetes.io/docs/user-guide/persistent-volumes/#storageclasses)
to the kafka StatefulSet declaration to enable automatic volume provisioning.

Alternatively create [PV](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistent-volumes)s and [PVC](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims)s manually. For example in Minikube.

```
kubectl create -f gce-volume-claims/gce-storage-class.yaml
```

## Set up Zookeeper StatefulSet

There is a Zookeeper+StatefulSet [blog post](http://blog.kubernetes.io/2016/12/statefulset-run-scale-stateful-applications-in-kubernetes.html) and [example](https://github.com/kubernetes/contrib/tree/master/statefulsets/zookeeper),
but it appears tuned for workloads heavier than Kafka topic metadata.

The Kafka book (Definitive Guide, O'Reilly 2016) recommends that Kafka has its own Zookeeper cluster,
so we use the [official docker image](https://hub.docker.com/_/zookeeper/)
but with a [startup script change to guess node id from hostname](https://github.com/solsson/zookeeper-docker/commit/df9474f858ad548be8a365cb000a4dd2d2e3a217).

If you lose your zookeeper cluster, kafka will be unaware that persisted topics exist.
The data is still there, but you need to re-create topics. So , we run in `StatefulSet`.

Zookeeper runs as a [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) with persistent storage:
```
kubectl create -f zookeeper/zoo-headless-svc.yml
kubectl create -f zookeeper/zoo-svc.yml
kubectl create -f zookeeper/zoo-stateful.yml
```

## Start Kafka

Assuming you have your PVCs `Bound`, or enabled automatic provisioning (see above), go ahead and:

```
kubectl create -f kafka/broker-headless-svc.yml
kubectl create -f kafka/kafka-svc.yml
kubectl create -f kafka/kafka-stateful.yml
```

You might want to verify in logs that Kafka found its own DNS name(s) correctly. Look for records like:
```
kubectl logs kafka-0 | grep "Registered broker"
# INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT -> EndPoint(kafka-0.broker.kafka.svc.cluster.local,9092,PLAINTEXT)
```

## Start Kafka-manager

As the image is too old , you should add the cluster manually . 

Cluster Zookeeper Hosts = `zookeeper:2181`

```
kubectl create -f kafka-manager/kafka-manager-svc.yml
kubectl create -f kafka-manager/kafka-manager.yml
```

## Automated test, while going chaosmonkey on the cluster

This is WIP, but topic creation has been automated. Note that as a [Job](http://kubernetes.io/docs/user-guide/jobs/), it will restart if the command fails, including if the topic exists :(
```
kubectl create -f test/11topic-create-test1.yml
```

Pods that keep consuming messages (but they won't exit on cluster failures)
```
kubectl create -f test/21consumer-test1.yml
```

## Teardown & cleanup

Testing and retesting... delete the namespace. PVs are outside namespaces so delete them manually.
```
kubectl delete namespace kafka
```

## Problems

1. As the Kafka runs in Kubernetes , the borker-list is just like below, so it may not provide external services .
```
2017/05/16 19:45:58 client/brokers registered new broker #2 at kafka-2.broker.kafka.svc.cluster.local:9092
2017/05/16 19:45:58 client/brokers registered new broker #1 at kafka-1.broker.kafka.svc.cluster.local:9092
2017/05/16 19:45:58 client/brokers registered new broker #0 at kafka-0.broker.kafka.svc.cluster.local:9092
```
