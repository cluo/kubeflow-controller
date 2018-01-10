## Kubeflow Controller Design Document

## Overview

Kubeflow controller is similar to [Kubernetes job controller](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/job/job_controller.go), which is an internal resource controller. In Kubernetes, a controller is a control loop that watches the shared state of the cluster through the apiserver and makes changes attempting to move the current state towards the desired state.

## Behavior

The controller initializes three resource informers: TFJob, Pod, and Service, and registers event handlers for the informers. Every time a pod/service/tfjob is created, updated, or deleted, the controller checks if actions should be taken to move the current state towards the desired state.

A typical distributed TFJob contains one or more PS, one or more workers, and every PS/worker is run in a pod, and exposed to other PS/workers via a service:

```
TFJob
  PS:
    - ps-0
      - pod/ps-0
      - svc/ps-0
    - ps-1
      - pod/ps-1
      - svc/ps-1
  worker:
    - worker-0
      - pod/worker-0
      - svc/worker-0
    - worker-1
      - pod/worker-1
      - svc/worker-1
    ...
```

And pods and services created by the TFJob are claimed to be controlled and created by the TFJob in Kubernetes.

### A typical description for TFJob

This is a typical description for TFJob, which is printed by `kubectl describe`:

```
Name:         dist-training-job
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  kubeflow.caicloud.io/v1alpha1
Kind:         TFJob
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-01-03T06:46:17Z
  Generation:          0
  Resource Version:    560
  Self Link:           /apis/kubeflow.caicloud.io/v1alpha1/namespaces/default/tfjobs/dist-training-job
  UID:                 cceb67c9-f051-11e7-99ac-484d7e9d305b
Spec:
  Runtime ID:  nbbbg
  Tf Replica Spec:
    Replicas:  2
    Template:
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Index:                     1
          Job _ Type:                PS
          Kubeflow . Caicloud . Io:  true
          Runtime _ Id:              nbbbg
          Tf _ Job _ Name:           dist-training-job
      Spec:
        Containers:
          Args:
            --worker_hosts=dist-training-job-worker-0-d88wg:2222,dist-training-job-worker-1-hwl26:2222,dist-training-job-worker-2-jx7g7:2222,dist-training-job-worker-3-gdb66:2222
            --ps_hosts=dist-training-job-ps-0-nlfdp:2222,dist-training-job-ps-1-jw7c4:2222
            --job_name=ps
            --task_index=1
          Command:
            python
            /workdir/mnist_replica.py
          Image:  <sensitive data>
          Name:   tensorflow
          Ports:
            Container Port:  2222
          Resources:
          Volume Mounts:
            Mount Path:  /workdir
            Name:        workdir
        Volumes:
          Host Path:
            Path:     /home/ist/go/src/github.com/caicloud/kubeflow-controller/examples/workdir
            Type:     Directory
          Name:       workdir
    Tf Replica Type:  PS
    Replicas:         4
    Template:
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Index:                     3
          Job _ Type:                Worker
          Kubeflow . Caicloud . Io:  true
          Runtime _ Id:              nbbbg
          Tf _ Job _ Name:           dist-training-job
      Spec:
        Containers:
          Args:
            --worker_hosts=dist-training-job-worker-0-d88wg:2222,dist-training-job-worker-1-hwl26:2222,dist-training-job-worker-2-jx7g7:2222,dist-training-job-worker-3-gdb66:2222
            --ps_hosts=dist-training-job-ps-0-nlfdp:2222,dist-training-job-ps-1-jw7c4:2222
            --job_name=worker
            --task_index=3
          Command:
            python
            /workdir/mnist_replica.py
          Image:  <sensitive data>
          Name:   tensorflow
          Ports:
            Container Port:  2222
          Resources:
          Volume Mounts:
            Mount Path:  /workdir
            Name:        workdir
        Restart Policy:  OnFailure
        Volumes:
          Host Path:
            Path:     /home/ist/go/src/github.com/caicloud/kubeflow-controller/examples/workdir
            Type:     Directory
          Name:       workdir
    Tf Replica Type:  Worker
Status:
  Conditions:  <nil>
  Phase:       Succeeded
  Reason:
  Tf Replica Statuses:
    State:
    Tf Replicas States:
      Succeeded:  4
    Type:         Worker
    State:
    Tf Replicas States:
      Running:  2
    Type:       PS
Events:
  Type    Reason            Age                From                 Message
  ----    ------            ----               ----                 -------
  Normal  SuccessfulCreate  36s                kubeflow-controller  Created service: dist-training-job-worker-0-d88wg
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created service: dist-training-job-worker-1-hwl26
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created service: dist-training-job-worker-2-jx7g7
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created service: dist-training-job-worker-3-gdb66
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created service: dist-training-job-ps-0-nlfdp
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created service: dist-training-job-ps-1-jw7c4
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created pod: dist-training-job-2j9br
  Normal  SuccessfulCreate  35s                kubeflow-controller  Created pod: dist-training-job-pm85l
  Normal  SuccessfulCreate  34s                kubeflow-controller  Created pod: dist-training-job-5rgvh
  Normal  SuccessfulCreate  32s (x3 over 34s)  kubeflow-controller  (combined from similar events): Created pod: dist-training-job-n48d7
```

The TFJob could get statuses of all workers and PS and report in the description. But there are some known issues because we do not implement deep copy in the first version. And we will discuss later in known issues section.

### A typical description for pod created by the TFJob

This is a typical description for pod created by the TFJob, which is printed by `kubectl describe`:

```
Name:           dist-training-job-z8669
Namespace:      default
Node:           127.0.0.1/127.0.0.1
Start Time:     Thu, 04 Jan 2018 14:31:45 +0800
Labels:         index=2
                job_type=Worker
                kubeflow.caicloud.io=true
                runtime_id=vg2v4
                tf_job_name=dist-training-job
Annotations:    kubernetes.io/created-by={"kind":"SerializedReference","apiVersion":"v1","reference":{"kind":"TFJob","namespace":"default","name":"dist-training-job","uid":"eea48177-f118-11e7-8e8f-484d7e9d305b","ap...
Status:         Pending
IP:
Created By:     TFJob/dist-training-job
Controlled By:  TFJob/dist-training-job
Containers:
  tensorflow:
    Container ID:
    Image:         <sensitive data>
    Image ID:
    Port:          2222/TCP
    Command:
      python
      /workdir/mnist_replica.py
    Args:
      --worker_hosts=dist-training-job-worker-0-nxv2x:2222,dist-training-job-worker-1-gblzb:2222,dist-training-job-worker-2-wch6f:2222,dist-training-job-worker-3-45p86:2222
      --ps_hosts=dist-training-job-ps-0-n2k99:2222,dist-training-job-ps-1-fx926:2222
      --job_name=worker
      --task_index=2
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Mounts:
      /workdir from workdir (rw)
Conditions:
  Type           Status
  Initialized    True
  Ready          False
  PodScheduled   True
Volumes:
  workdir:
    Type:  HostPath (bare host directory volume)
    Path:  /home/ist/go/src/github.com/caicloud/kubeflow-controller/examples/workdir
```

Most of the configs are user-specified in [dist.yml](https://github.com/caicloud/kubeflow-controller/blob/master/examples/tfjob/dist.yml), and the labels and arguments are generated by the controller. The label is used to expose port, and the arguments is to hide the cluster spec from user code for ease of use.

## Implementation

Lifecycle management is the main responsibility, and the logic is in `pkg/controller/controller.go`.

We use `SharedInformers` to ListWatch the resource TFJob, and register specify event handlers to fill the workqueue with TFJob creations, updates and deletions. Then process one item from the workqueue at a time:

- Check if the TFJob is finished
- Check the cache which is updated when the controller watches pods or services state changed, and try to make the TFJobs's current state be more like the desired state
- Update the status of the TFJob to apiserver
- Do cleanup if the TFJob is finished during the process

For example, if there is a new TFJob `dist-job` created, and the event handler [`AddFunc`](https://github.com/caicloud/kubeflow-controller/blob/2c6567508ed05804091758ad25651d4937c1c345/pkg/controller/controller.go#L136) enqueues the tfjob to workqueue and it is handled in [`syncHandler`](https://github.com/caicloud/kubeflow-controller/blob/2c6567508ed05804091758ad25651d4937c1c345/pkg/controller/controller.go#L261). The function decides if we should take action according to the TTLCache. If there is no enough worker/PS, the controller will create new pods and services in [`manageTFJob`](https://github.com/caicloud/kubeflow-controller/blob/2c6567508ed05804091758ad25651d4937c1c345/pkg/controller/controller.go#L356). And because we do not implement deep equal before, we update the TFJob status regardless of the TFJob.

[`pkg/tensorflow`](https://github.com/caicloud/kubeflow-controller/tree/2c6567508ed05804091758ad25651d4937c1c345/pkg/tensorflow) package checks the status of the TFJob and tell the controller what to do to move to desired state.

## Progress

Now the controller could handle the normal cases of local and distributed training jobs. But it fails to handle the failures of pods or services, and it has some known issues. And this controller is in active development and may be changed frequently.

## Known issues

There are some known bugs in [caicloud/kubeflow-controller/issues](https://github.com/caicloud/kubeflow-controller/issues):

### We can not recover from the failure of pods

https://github.com/caicloud/kubeflow-controller/issues/36

For example, we request to create 4 workers 2 PSs, and the events will be:

```
[
    Event{
        Type: AddWorker,
        Number: 4,
    },
    Event{
        Type: AddPS,
        Number: 2,
    },
]
```

If there is a woker with `task_index=3` failed accidentally, we could not create a new worker with `task_index=3` since the controller will receives events:

```
[
    Event{
        Type: AddWorker,
        Number: 1,
    },
]
```

The controller knows that it should create a new worker but it has no idea to create a new worker with `task_index=3`.

We need to add some info to the event or design a more graceful mechanism to let the controller know what happened.

### Import deep copy to solve argument problem

https://github.com/caicloud/kubeflow-controller/issues/28

Now we have to set different arguments for each worker and ps since there is an arg task_index, and we need uses deep copy to make sure that the argument changes will not affect others.

We decide to develop based on Kubernetes 1.7 and now we migrate to 1.8 then I think the deep copy code could be generated by k8s.io/code-generator.
