# Affinity and anti-affinity

Affinity/anti-affinity is similar to nodeSelector but allows us to do more complex scheduling than the nodeSelector. nodeSelector can only apply the conditions on nodes while affinity/anti-affinity can also be applied to Pods. Also, unlike nodeSelector, affinity/anti-affinity rules does not have to be *hard* rules, which means a Pod can still be scheduled even if the rules are not met.

Kubernetes can do node affinity and pod affinity. Node affinity is similar to nodeSelector. Pod affinity allows to create rules that schedules Pods while taking other Pods into account. Those rules only relevant during scheduling. Once a Pod is scheduled, it needs to be killed and recreated to apply the rules again.

## Node Affinity

There are two types for node affinity:
1. requiredDuringSchedulingIgnoredDuringExecution
2. preferredDuringSchedulingIgnoredDuringExecution

The first is a hard requirement like nodeSelector (the cluster has to meet the rule to schedule a Pod) and the second is a soft requirement.

## Schedule Pods using `requiredDuringSchedulingIgnoredDuringExecution`

Please refer to [this](./deploy/node-affinity-required.yaml) file for Pod deployment. To schedule this Pod, we can see that it requires `hardware` in `high` *hard* requirement.

Before creating the Pods, let's check the node labels first.
```shell
$ kubectl get pods -o wide
NAME           STATUS   ROLES    AGE    VERSION   LABELS
192.168.1.51   Ready    <none>   116d   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.1.51
192.168.1.52   Ready    <none>   97d    v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.1.52
192.168.1.61   Ready    <none>   32d    v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.1.61
192.168.1.62   Ready    <none>   32d    v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.1.62
```

There is no node labeled as `hardware=high` and therefore, no Pod should be scheduled. To verify this, run:
```shell
$ kubectl create -f node-affinity-required.yaml

$ kubectl get pods -o wide
...
node-affinity-77968676d9-dxpgd    0/1     Pending   0          <invalid>   <none>        <none>         <none>           <none>
node-affinity-77968676d9-ndzqh    0/1     Pending   0          <invalid>   <none>        <none>         <none>           <none>
node-affinity-77968676d9-wxdjn    0/1     Pending   0          <invalid>   <none>        <none>         <none>           <none>
...
```

It is as expexted that the Pods are pending in creation status.

Next, we add `hardware=high` label to `192.168.1.51` node
```shell
$ kubectl label nodes 192.168.1.51 hardware=high
node/192.168.1.51 labeled
$ kubectl get pods -o wide
...
node-affinity-77968676d9-dxpgd    0/1     ContainerCreating   0          <invalid>   <none>         192.168.1.51   <none>           <none>
node-affinity-77968676d9-ndzqh    1/1     Running             0          <invalid>   172.30.81.2    192.168.1.51   <none>           <none>
node-affinity-77968676d9-wxdjn    1/1     Running             0          <invalid>   172.30.81.10   192.168.1.51   <none>           <none>
...
```

All Pods are scheduled to `192.168.1.51`. Let's apply the same label to `192.168.1.52` as well: 
```shell
$ kubectl label nodes 192.168.1.51 hardware=high`
node/192.168.1.52 labeled

$ kubectl get po
```

After applying the label to `192.168.1.52` the Pods are still running on `192.168.1.51`, which is also as expected because affinith/anti-affinity is only relevant during Pod creation.

Now, delete the Pod and recreate them:
```shell
$ kubectl delete -f node-affinity-required.yaml
deployment.extensions "node-affinity" deleted

$ kubectl create -f node-affinity-required.yaml
deployment.extensions/node-affinity created

$ kubectl get pods -o wide
node-affinity-77968676d9-6nzxp    1/1     Running   0          21s    172.30.10.4   192.168.1.52   <none>           <none>
node-affinity-77968676d9-dxqpv    1/1     Running   0          21s    172.30.81.2   192.168.1.51   <none>           <none>
node-affinity-77968676d9-qnb4p    1/1     Running   0          21s    172.30.81.5   192.168.1.51   <none>           <none>
```

The Pods are scheduled on both `51` and `52`.