## kube-batch

gang scheduler介绍：一个kube-batch作业（kube-batch job）可能有多个pods, 这些pods要不全部执行，要不一个都不执行。

### 1. 名词

 + k8s job：一些pod集合

 + kube-batch task: pod，一个kube-batch task就是一个pod

 + kube-batch podGroup: 一组pod，这是kube-batch自定义的crd，主要用来实现gang scheduler。通过设置podgroup的minNumber，达到每次调度要么执行minNumber个pods.要么一个都不执行。

 + kube-batch job: 一组pod，指向某podGroup的所有pod(一个kube-batch job就是一个podGroup，见下图)，可以对应一个或多个k8s job

