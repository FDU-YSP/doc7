使用kubeflow结合kubernetes进行大规模分布式训练时，由于AI场景下对任务的调度需要: all or nothing, multi-tenant task queue, task priority, preemption,gpu affinity等条件，但是kubernetes默认的调度器对这些条件还没有完全的支持，幸运的是kubernetes社区孵化了一个适合训练的调度器kube-batch。

### AI 场景下的调度有何不同？

- gang-schedule(all or nothing)
- multi-tenant task queue
- task priority
- preemption
- gpu affinity

第一点比较重要的就是`gang-schedule`。`gang-schedule`是什么概念? 用用户提交一个 batch job, 这个batch job 包含100个任务，要不这100个任务全部调度成功，要么一个都调度不成功。这种`all or nothing`调度场景，就被称作:`gang-schedule`，通常用于集群资源不足的场景，比如 AI 场景下调用GPU资源。

第二点就是AI场景需要提供多住户的任务队列。因为GPU资源是非常宝贵的，经常会出现集群资源不可用的情况。当不可用时，需要有租户或者任务的优先级，并且每个租户需要有对应的优先级队列，队列里面又有任务优先级，设计不同的优先级住户之间的资源抢占，或者同一个租户不同优先级任务的资源抢占。另外`GPU Affinity`的调度能力，可以提高模型的训练性能，也是scheduler需要考虑的事情。

### kube-batch 调度器介绍

`kube-batch`是一个为kubernetes实现批量任务调度的一个调度器，主要用于`机器学习`，`大数据`，`HPC`等场景。

![](.\img\3.png)

为了使用`gang-scheduling`，需要在kubernetes集群提前安装`kube-batch`调度器。具体的安装步骤查看官方教程: [kube-batch install](https://github.com/kubernetes-sigs/kube-batch/blob/master/doc/usage/tutorial.md)

安装完成`kube-batch`并且测试没有问题，便可以在基于Kubeflow的分布式训练中指定该调度器(kube-batch)。

`注意`

```

Take tf-operator for example, enable gang-scheduling in tf-operator by setting true to 
--enable-gang-scheduling flag.
```

好了，所有的准备工作做完之后，可以使用kube-batch去调度一个任务，为了让任务使用kube-batch，还需要在yaml文件中指定`schedulerName`参数为kube-batch。下面是详细的例子:

```yaml
apiVersion: "kubeflow.org/v1beta1"
kind: "TFJob"
metadata:
  name: "tfjob-gang-scheduling"
spec:
  tfReplicaSpecs:
    Worker:
      replicas: 1
      template:
        spec:
          schedulerName: kube-batch
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=gpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-gpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            resources:
              limits:
                nvidia.com/gpu: 1
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
    PS:
      replicas: 1
      template:
        spec:
          schedulerName: kube-batch
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=cpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            resources:
              limits:
                cpu: '1'
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
```

### 总结

在设计AI Paas平台时，除了功能跑起来只是一个开始，在测试过程中随之而来的，网络，存储，调度等等用影响性能的问题，都需要花费大量的精力去解决，基于`kube-batch`便可以基本解决在调度过程中，kubernetes默认调度器的不足，来达到`all or nothing`的目的。