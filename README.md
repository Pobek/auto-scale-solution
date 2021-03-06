# Auto Scaling Solution

This repository contains the solution to the auto-scaling incident exercise..

## Problem

Our production AWS k8s cluster is running on different instance groups, all of them are spot instance based.

During a scaling event, we were unable to acquire new spot instances due to their availability dropping to 0 in the regions we reside in (Although our Max Price was set to the on demand value)

This event caused a major degradation to our performance due to the fact our services were not able to deal with the increasing request throughput.

## Task

design a solution that can deal with the described situation, while maintaining high availability and being as cost effective as possible

## Solution

The solution to the problem will take a few points as assumptions:

1. The current spot instance groups have 'working' nodes and didn't terminate. (meaning the current instance groups node number remains the same, just cannot request more).
2. The kubernetes cluster was created using kops tool.
3. State bucket is already created.
4. IAM roles are already created.

### Base lines

#### Cluster

- A cluster resource should be created :

  ```bash
  kops create cluster \
  --name evya.test.cluster \
  --state s3://evya-cluster-state \
  --cloud aws \
  --master-size t3.large \
  --master-count 1 \
  --master-zones us-east-1a \
  --node-size t3.large \
  --node-count 1 \
  --zones us-east-1a,us-east-1b,us-east-1c \
  --ssh-public-key ~/.ssh/evya_cluster_rsa.pub \
  ```

  - Once verified with `kops update cluster evya.test.cluster` that what we want to created is correct, run `kops update cluster evya.test.cluster --yes`

#### Instance groups

- Two spot instance groups should be configured :

  - Spot instance group [spot-1](kops-manifests/spot-1.yaml)

    ```yaml
    apiVersion: kops/v1alpha2
    kind: InstanceGroup
    metadata:
      labels:
        kops.k8s.io/cluster: evya.test.cluster
        k8s.io/cluster-autoscaler/enabled: true
        k8s.io/cluster-autoscaler/evya.test.cluster: owned
      name: spot-1
    spec:
      image: kope.io/k8s-1.16-debian-stretch-amd64-hvm-ebs-2020-01-17
      machineType: t3.large
      maxPrice: $(MAX_PRICE)
      maxSize: 3
      minSize: 1
      nodeLabels:
        kops.k8s.io/instancegroup: spot-1
        node-type/spot: "true"
      role: Node
      subnets:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    ```

  - Spot instance group [spot-2](kops-manifests/spot-2.yaml)

    ```yaml
    apiVersion: kops/v1alpha2
    kind: InstanceGroup
    metadata:
      labels:
        kops.k8s.io/cluster: evya.test.cluster
        k8s.io/cluster-autoscaler/enabled: true
        k8s.io/cluster-autoscaler/evya.test.cluster: owned
      name: spot-2
    spec:
      image: kope.io/k8s-1.16-debian-stretch-amd64-hvm-ebs-2020-01-17
      machineType: m5.large
      maxPrice: $(MAX_PRICE)
      maxSize: 3
      minSize: 1
      nodeLabels:
        kops.k8s.io/instancegroup: spot-2
        node-type/spot: "true"
      role: Node
      subnets:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    ```

### Proposed solution

The proposed solution is to create a third instance group, but rather to define it for 'spot', we define it for on-demand but only as our fallback. Meaning, when an instance group needs to be scaled out, it will first try to scale a spot instance group and if cannot, will scale on-demand instance group.

The proposed solution is taking a usage of [Kubernetes Cluster AutoScaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) application.

To simply explain, the application takes a few arguments as configuration, explaining to the application which AWS ASG (instance groups) should it be looking for. The application will also listen to kubernetes events regarding deployments / pods and will increase / decrease the cluster size accordingly.

#### on-demand instance group

The third on-demand instance group '[on-demand](kops-manifests/on-demand.yaml)':

```yaml
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: evya.test.cluster
    k8s.io/cluster-autoscaler/enabled: true
    k8s.io/cluster-autoscaler/evya.test.cluster: owned
  name: nodes
spec:
  image: kope.io/k8s-1.16-debian-stretch-amd64-hvm-ebs-2020-01-17
  machineType: t3.large
  maxSize: 3
  minSize: 0
  nodeLabels:
    kops.k8s.io/instancegroup: on-demand
    node-type/on-demand: "true"
  role: Node
  subnets:
  - us-east-1a
  - us-east-1b
  - us-east-1c
  taints:
  - node-type/on-demand=true:PreferNoSchedule
```

#### Cluster AutoScaler configuration

The AutoScaler app will be configured as follows:

- Cloud provider : AWS
- Skip nodes with local storage: False
- Expander type: Priority
- Node groups: auto discovered:
  - Tags: 
    - k8s.io/cluster-autoscaler/enabled
    - k8s.io/cluster-autoscaler/evya.test.cluster
- balance-similar-node-groups: True

Before deploying the autoscaler app, a priority config map needs to be created to let the autoscaler know the priority of each instance group '[priority-expander-cm](cluster-autoscaler/priority-expander-cm.yaml)':

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cluster-autoscaler-priority-expander
    namespace: kube-system
  data:
    priorities: |-
      10: 
        - .*nodes.*
      20: 
        - .*spot.*
  ```

> The priority-expander configmap lets the autoscaler know the priority per instance group that is matched with the regex. The highest value wins and it will be auto-scaled first and unless it cannot, it won't scale the others.

AutoScaler's arguments to be passed (entire yaml file can be seen on [autoscaler.yaml](cluster-autoscaler/autoscaler.yaml)):

  ```bash
  command:
    - ./cluster-autoscaler
    - --v=4
    - --stderrthreshold=info
    - --cloud-provider=aws
    - --skip-nodes-with-local-storage=false
    - --expander=priority
    - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/evya.test.cluster
    - --balance-similar-node-groups
  ```

After configuring the auto-scaler, whenever an update occurs on the deployment's replica count and the pod is in "Pending" state due to lack of resources, the auto-scaler will try to scale either of the spot instance-groups node count first and if cannot, will scale the on-demand instance group node count.

#### Spot Termination Handler

Even though we deployed an auto-scaler to automatically scale the instance-group's count according to our workload, when running on 'spot-instances' we run into a risk of our instance being terminated by AWS. AWS provides a two minute notice about reclaiming the 'spot-instances' and we need to look after it.

To do so, the solution uses [k8s-spot-termination-handler](https://github.com/helm/charts/tree/master/stable/k8s-spot-termination-handler) helm2 chart.

There is no need to set any custom values at this point.

Install command:

```bash
helm install stable/k8s-spot-termination-handler --namespace kube-system --name spot-term-handler
```

The `k8s-spot-termination-handler` deploys a DaemonSet (one pod per node) and will query the EC2 spot instance termination endpoint to see if the node he is currently on is going to be shut down. If so, the application will gracefully start to shut down all of the pods on the current node to allow them to be provisioned onto a different node.

#### Scaling back to spot when ready

Even though the solution defines a way to auto-scale into on-demand instances when 'spot' instances aren't available, it won't check to see if any 'spot' instances are back in the pool unless there is a change in the amount of replications.

To make sure we always be on the 'spot' instances and only be on the 'on-demand' instances when it is a **must**, the solution will use an application called [k8s-spot-rescheduler](https://github.com/pusher/k8s-spot-rescheduler)

Taken from their github: 

```
K8s Spot rescheduler is a tool that tries to reduce load
on a set of Kubernetes nodes.
It was designed with the purpose of moving Pods scheduled
on AWS on-demand instances to AWS spot instances
to allow the on-demand instances to be safely scaled down
```

Because the solution created a new instance group just for 'on-demand' instance types, added a taint of `PreferNoSchedule` and node labels that allow us to differentiate between 'spot' instances and 'on-demand' instances, we can configure `k8s-spot-rescheduler` with the following config to allow it to work (entire deployment file can be seen on [rescheduler.yaml](spot-rescheduler/rescheduler.yaml)):

  ```
  command:
    - k8s-spot-rescheduler
    - -v=2
    - --running-in-cluster=true
    - --namespace=kube-system
    - --housekeeping-interval=10s
    - --node-drain-delay=10m
    - --pod-eviction-timeout=2m
    - --max-graceful-termination=2m
    - --listen-address=0.0.0.0:9235
    - --on-demand-node-label=node-type/on-demand
    - --spot-node-label=node-type/spot
  ```

Once deployed, `k8s-spot-rescheduler` will look to see if it is possible to migrate pods that exists on the 'on-demand' instance group, to the 'spot' instance group. If so, will start to drain the 'on-demand' node and the default kubernetes schedule will start to work. If not, will do nothing.