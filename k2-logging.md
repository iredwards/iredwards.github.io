---
title: K2 Logging
permalink: k2-logging.html
toc: false
author: red
---

The K2 logging tools provide a centralized logging solution for a production grade Kubernetes cluster. This system ensures events are safely pulled from pods, enriched with Kubernetes metadata, saved in a data store, and made available for visualizing and querying.

## Installation
Use [Helm](https://github.com/kubernetes/helm) to install the logging system in your cluster. Helm is installed with K2. If needed you can install Helm manually, as described [here](https://github.com/kubernetes/helm/blob/master/docs/install.md).

The following commands install the logging system:
```
$ helm repo add cnct http://atlas.cnct.io
$ helm install cnct/central-logging
```

Alternatively you can install the logging system by adding the following to your [K2](https://github.com/samsung-cnct/k2) configuration template:

``` clusterServices:
repos:
-
name: atlas
url: http://atlas.cnct.io
services:
-
name: central-logging
repo: atlas
chart: central-logging
version: 0.1.0
```

## Logging system components
The following assets describe components in the centralized logging system.
Architectural blurb here.


### Log Transport Bus  
Kafka is used for transporting logs, and has three separate systems:

-   [ZooKeeper](https://zookeeper.apache.org/) which is a quorum service and is a required subsystem of Kafka. There is an example StatefulSet deployment in the kubernetes contrib repository that we will use as the basis of a chart
-   [Kafka](http://kafka.apache.org/) which is a messaging system that we will use for transporting logs, decoupling input systems from output systems, replayability and several other functions. This will be a StatefulSet modeled after the ZooKeeper StatefulSet. We are using a StatefulSet with Persistent Volume Claims and deterministic broker ids to provide data resiliency in the face of pod failures. This will decrease performance so it is important to keep an eye on performance as we go forward
-   [Kafka Monitor](https://github.com/linkedin/kafka-monitor) provides continuous and built-in monitoring of Kafka performance. This is deployed on a single pod.

#### ZooKeeper

ZooKeeper is used for communicating state between the brokers.  

Kafka uses ZooKeeper for _service discovery_ - the process of learning the location of services, and _consensus_ - the process of getting several nodes to agree on something. In a Kafka cluster, service discovery helps the brokers find each other and know who’s in the cluster; and consensus helps the brokers elect a cluster controller, know what partitions exist, where they are, if they’re a leader of a partition or if they’re a follower and need to replicate, and so on.

Deployment configs: https://github.com/kubernetes/contrib/tree/master/pets/zookeeper

#### Kafka

Kafka is a robust messaging system. It provides durability for received messages, message replay, decoupling of source and destination systems, and allows for fanout. There is a cost in CPU, memory and disk in using this system but it is well worth the advantages to have this abstraction available. We have chosen a StatefulSet for this deployment over a DaemonSet because we believe the tradeoff in degraded performance for robustness and reliability is a worthwhile trade. We may be wrong so we will need to be continuously taking metrics and deciding what's best on a case by case basis. A daemon set chart will be available in the near future but not in the v0.1 release.

Deployment configuration: [https://github.com/samsung-cnct/kafka-kubernetes](https://github.com/samsung-cnct/kafka-kubernetes). This may be merged into the upstream kubernetes/contrib repo.

#### Kafka Monitor

Kafka Monitor is an open-sourced Kafka monitoring tool developed by Linkedin (the authors of Kafka). It is a framework to implement and execute long-running Kafka system tests. It monitors Kafka by putting messages into a known topic and measuring certain aspects of message delivery.

Deployment configuration files are located in: https://github.com/samsung-cnct/kafka-monitor-kubernetes. The Docker image creation scripts have been pushed upstream into https://github.com/linkedin/kafka-monitor.

Github Repo: [https://github.com/samsung-cnct/kafka-kubernetes](https://github.com/samsung-cnct/kafka-kubernetes)  
Monitor Github Repo: [https://github.com/samsung-cnct/kafka-monitor-kubernetes](https://github.com/samsung-cnct/kafka-monitor-kubernetes)  
Image: [https://quay.io/repository/samsung_cnct/kafka-petset](https://quay.io/repository/samsung_cnct/kafka-petset)  
Monitor Image: [https://quay.io/repository/samsung_cnct/kubernetes-kafka-monitor](https://quay.io/repository/samsung_cnct/kubernetes-kafka-monitor)  
Helm Chart: [https://github.com/samsung-cnct/k2-charts/tree/master/kafka](https://github.com/samsung-cnct/k2-charts/tree/master/kafka)

### Node Collection and Processing  
The Node Collector is a [Fluentd](http://www.fluentd.org/) deployment that receives logs from Kafka, lets you further process or label them, and then send them to the storage systems you use. A FluentD daemonset is responsible for collecting logs from applications on pods. The Fluentd processor filters the logs so you can monitor the health of your system at a pod level. Fluentd is an open source data collector that offers a flexible plugin system to configure different input and output options.

The node collector is implemented as a Kubernetes [deployment](http://kubernetes.io/docs/user-guide/deployments/).

Github Repo: [https://github.com/samsung-cnct/k2-logging-central-fluentd](https://github.com/samsung-cnct/k2-logging-central-fluentd)  
Image: [https://quay.io/repository/samsung_cnct/fluentd-central](https://quay.io/repository/samsung_cnct/fluentd-central)  
Helm Chart: [https://github.com/samsung-cnct/k2-charts/tree/master/central-fluentd](https://github.com/samsung-cnct/k2-charts/tree/master/central-fluentd)

## Queryable Datastore: ElasticSearch  
To achieve a highly available, fault tolerant cloud native architecture, there is a need to tolerate a fluctuating member set, attachable drives for data, and an external coordinator for member management. This is the motivation for a multi-configuration cluster based on Kubernetes StatefulSets. The master configuration will spin up a fixed number of nodes that are used only for master activities, e.g. shard creation, quorum participation, stats gathering, etc. A second configuration defining data and ingest nodes can be scaled as necessary, subject to Elasticsearch constraints. This provides a balance between Elasticsearch's scalability, and the complexity of handling networking partitions of the master set.

General member discovery is done using fabric.io's Kubernetes discovery plugin.

### Architecture Diagram

![ES Architecture](images/ES_Architecture.png)

### Master Pods

Master pods are modeled as a StatefulSet (formerly PetSet) because it provides a well managed set of pods that have some partition resiliency built in. There will be three or five pods in this set. That number should be set at cluster creation and then not modified for the life of the cluster. Changing the pod count requires a reconfiguration of the running pods, which requires manual intervention.

The quantity of data the master pods handle rather small and can be replicated quickly from a peer. However, a simple deployment is not appropriate because it is important that a network partition doesn't create a duplicate pod. That can happen if a single pod becomes isolated on the network from the other two pods, and the single pod is on the side of the network with the control plane. The control plane will notice that there is only one pod running and attempt to spin up two more. That operation must fail because the isolated pods still exist, and are in quorum and accepting writes on the other side of the partition. The control plane will not be able to create new duplicate pods because the persistent volume claims for the named pods will still exist and be bound to the existing pods. If the existing pods are no longer able to talk to the block store they will quickly drop into a failed state, and then having the control plane spin up new pods is the correct behavior.

### Data Pods

Data pods are modeled as a StatefulSet because the data-pod name affinity is significant; Kubernetes should quickly reschedule pods that go down due to node issues or operator actions. These pods are free to be scaled out as large as required or desired. They will not take part in any quorum calculations or other master node activities. If the number of pods in this set is reduced, care should be taken that data be allowed to replicate after a pod is terminated.  

Github Repo: [https://github.com/samsung-cnct/k2-charts/tree/master/elasticsearch](https://github.com/samsung-cnct/k2-charts/tree/master/elasticsearch)  
Image: [https://quay.io/repository/samsung_cnct/elasticsearch](https://quay.io/repository/samsung_cnct/elasticsearch)  
Helm Chart: [https://github.com/samsung-cnct/k2-charts/tree/master/elasticsearch](https://github.com/samsung-cnct/k2-charts/tree/master/elasticsearch)

**Data Visualization** Kibana  
The final destination of the logs in the central logging pipeline is [Kibana](https://www.elastic.co/products/kibana), an open source analytics and visualization platform designed to work with [Elasticsearch](https://www.elastic.co/products/elasticsearch). You can use Kibana to search, view, and interact with data stored in Elasticsearch indices.

To deploy Kibana, you will need a Kubernetes [service](https://kubernetes.io/docs/user-guide/services/) and a Kubernetes [deployment](http://kubernetes.io/docs/user-guide/deployments/).

Deployment configuration: [https://github.com/leahnp/central_logging_test](https://github.com/leahnp/central_logging_test)

  -   kibana-deployment.yaml
  -   kibana-service.yaml

Once up and running, the external IP for Kibana can be found using the command `kubectl <...> get svc -owide`.

To access Kibana through its external IP, use the username and password listed at the end of the file `/k2/cluster/<cluster-name>/admin.kubeconfig`.

Github Repo: [https://github.com/samsung-cnct/kibana-k2-dependencies](https://github.com/samsung-cnct/kibana-k2-dependencies)  
Image: [https://quay.io/repository/samsung_cnct/kibana](https://quay.io/repository/samsung_cnct/kibana)  
Helm Chart: [https://github.com/samsung-cnct/k2-charts/tree/master/kibana](https://github.com/samsung-cnct/k2-charts/tree/master/kibana)
