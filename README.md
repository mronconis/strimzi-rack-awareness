# Strimzi rack-awareness
This repository aim to provide an example Kafka CR for deploying a kafka cluster across multiple AZ/DC with dedicated nodes and rack-awareness.

The rack awareness feature spreads replicas of the same partition across different racks. This extends the guarantees Kafka provides for broker-failure to cover rack-failure, limiting the risk of data loss should all the brokers on a rack fail at once. The feature can also be applied to other broker groupings such as availability zones in EC2.

For further details see the following links:
- [strimzi-running-kafka-on-dedicated-nodes](https://strimzi.io/blog/2018/07/30/running-kafka-on-dedicated-nodes/)
- [strimzi-spreading-partition-replicas-across-racks](https://github.com/strimzi/strimzi-kafka-operator/blob/main/documentation/api/io.strimzi.api.kafka.model.common.Rack.adoc)
  
## Create K8s cluster
Create cluster:
```bash
kind create cluster --name k8s-cluster --config kind.yaml
```

To create a dedicated node, you have to first taint it. Taints are a feature of nodes which can be used to repel pods. Only pods which tolerate a given taint can be scheduled on such nodes. The nodes can be tainted using the kubectl tool:
```bash
# Kafka
kubectl taint nodes k8s-cluster-worker dedicated=kafka:NoSchedule
kubectl taint nodes k8s-cluster-worker2 dedicated=kafka:NoSchedule
kubectl taint nodes k8s-cluster-worker3 dedicated=kafka:NoSchedule
# Zk
kubectl taint nodes k8s-cluster-worker4 dedicated=zookeeper:NoSchedule
kubectl taint nodes k8s-cluster-worker5 dedicated=zookeeper:NoSchedule
kubectl taint nodes k8s-cluster-worker6 dedicated=zookeeper:NoSchedule
```

Verify nodes labels:
```bash
kubectl get nodes --show-labels=true
```

Output:
```
NAME                        STATUS   ROLES           AGE    VERSION   LABELS
k8s-cluster-control-plane   Ready    control-plane   3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-cluster-worker          Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=kafka,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker,kubernetes.io/os=linux,topology.kubernetes.io/zone=az1
k8s-cluster-worker2         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=kafka,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker2,kubernetes.io/os=linux,topology.kubernetes.io/zone=az2
k8s-cluster-worker3         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=kafka,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker3,kubernetes.io/os=linux,topology.kubernetes.io/zone=az3
k8s-cluster-worker4         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=zookeeper,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker4,kubernetes.io/os=linux,topology.kubernetes.io/zone=az1
k8s-cluster-worker5         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=zookeeper,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker5,kubernetes.io/os=linux,topology.kubernetes.io/zone=az2
k8s-cluster-worker6         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,dedicated=zookeeper,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker6,kubernetes.io/os=linux,topology.kubernetes.io/zone=az3
k8s-cluster-worker7         Ready    <none>          3h9m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-cluster-worker7,kubernetes.io/os=linux
```

Verify nodes taints:
```bash
kubectl describe nodes | grep -E "Name:|Taints:"
```

Output:
```
Name:               k8s-cluster-control-plane
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Name:               k8s-cluster-worker
Taints:             dedicated=kafka:NoSchedule
Name:               k8s-cluster-worker2
Taints:             dedicated=kafka:NoSchedule
Name:               k8s-cluster-worker3
Taints:             dedicated=kafka:NoSchedule
Name:               k8s-cluster-worker4
Taints:             dedicated=zookeeper:NoSchedule
Name:               k8s-cluster-worker5
Taints:             dedicated=zookeeper:NoSchedule
Name:               k8s-cluster-worker6
Taints:             dedicated=zookeeper:NoSchedule
Name:               k8s-cluster-worker7
Taints:             <none>
```

## Install Strimzi operator
Before deploying the Strimzi cluster operator, create a namespace called kafka:
```bash
kubectl create namespace kafka
```

Apply the Strimzi install files, including ClusterRoles, ClusterRoleBindings and some Custom Resource Definitions (CRDs):
```bash
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Follow the deployment of the Strimzi cluster operator:
```bash
kubectl get pod -n kafka --watch -o wide
```

## Deploy Kafka cluster
Create kafka cluster:
```bash
kubectl apply -f kafka.yaml -n kafka
```

Waiting for cluster to be ready:
```bash
kubectl wait kafka/kafka-cluster --for=condition=Ready --timeout=300s -n kafka
```

Verify where pods are scheduled:
```bash
kubectl get pod -n kafka --watch -o wide
```

Output:
```
NAME                                             READY   STATUS    RESTARTS   AGE     IP           NODE                  NOMINATED NODE   READINESS GATES
kafka-cluster-entity-operator-75b48cf649-cs6w6   2/2     Running   0          28s     10.244.1.3   k8s-cluster-worker7   <none>           <none>
kafka-cluster-kafka-0                            1/1     Running   0          3m18s   10.244.4.3   k8s-cluster-worker    <none>           <none>
kafka-cluster-kafka-1                            1/1     Running   0          3m18s   10.244.3.3   k8s-cluster-worker3   <none>           <none>
kafka-cluster-kafka-2                            1/1     Running   0          3m18s   10.244.5.3   k8s-cluster-worker2   <none>           <none>
kafka-cluster-zookeeper-0                        1/1     Running   0          5m37s   10.244.7.3   k8s-cluster-worker4   <none>           <none>
kafka-cluster-zookeeper-1                        1/1     Running   0          5m37s   10.244.6.3   k8s-cluster-worker6   <none>           <none>
kafka-cluster-zookeeper-2                        1/1     Running   0          5m37s   10.244.2.3   k8s-cluster-worker5   <none>           <none>
strimzi-cluster-operator-66b5ff8bbb-5pxxd        1/1     Running   0          7m48s   10.244.1.2   k8s-cluster-worker7   <none>           <none>
```