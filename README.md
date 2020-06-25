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
        group-type: "spot"
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
        group-type: "spot"
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
    instance-type: "on-demand"
  role: Node
  subnets:
  - us-east-1a
  - us-east-1b
  - us-east-1c
  taints:
  - instance-type=on-demand:PreferNoSchedule
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
