# Anatomy of a Pod

## Introduction

The Pod is the master piece of the Kubernetes cluster architecture.

The fundamental goal of Kubernetes is to help you manage your containers. The Pod is the minimal piece deployable in a Kubernetes cluster, containing one or several containers.

From the `kubectl` command line, you can run a pod containing a container as simply as running this command:

```shell
$ kubectl run --generator=run-pod/v1 nginx --image=nginx
```

By adding `--dry-run -o yaml` to the command, you can see the YAML template you would have to write to create the same Pod:

```shell
$ kubectl run --generator=run-pod/v1 nginx --image=nginx --dry-run -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Or, if you extremely simplify the template:

```yaml
-- simple.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

You can now start the Pod by using this template:
```
$ kubectl apply -f simple.yaml
```

Or, if you are a Go developer, you can use the `client-go` library to run the Pod (complete sources are on the `simple/` directory):

```go
pod := corev1.Pod{
    ObjectMeta: metav1.ObjectMeta{
        Name: "nginx",
    },
    Spec: corev1.PodSpec{
        Containers: []corev1.Container{
            {
                Name:  "nginx",
                Image: "nginx",
            },
        },
    },
}

podsClient := clientset.CoreV1().Pods("default")
_, err := podsClient.Create(&pod)
```

In this introduction, we have seen three ways to create a Pod:
- with `kubectl run`
- with `kubectl apply`
- with the `client-go` library.

The Pod created is ready for production ... if you are not very fussy. Otherwise, the Pod offers a long list of parameters to make it more production-ready. We will examine in this document all these parameters.

## Pod specs

Here is a classification of the Pod parameters.

- **Containers** parameters will define and parameterize more precisely each container of the Pod, whether it is a normal container (`Containers`) or an init container (`InitContainers`). The `ImagePullSecrets` parameter will help to download containers images from private registries.

- **Volumes** parameter (`Volumes`) will define a list of volumes that containers will be able to mount and share.

- **Scheduling** parameters will help you define the most appropriate Node to deploy the Pod, by selecting nodes by labels (`NodeSelector`), directly specifying a node name (`NodeName`),  
using `Affinity` and `Tolerations`, by selecting a specific scheduler (`ScedulerName`), and by requiring a specific runtime class (`RuntimeClassName`). They will also be used to prioritize a Pod over other Pods (`PriorityClassName`and `Priority`).

- **Lifecycle** parameters will help define if a Pod should restart after termination (`RestartPolicy`) and fine-tune the periods after which processes running in the containers of a terminating pod are killed (`TerminationGracePeriodSeconds`) or after which a running Pod will be stopped if not yet terminated (`ActiveDeadlineSeconds`). They also help define readiness of a pod (`ReadinessGates`).

- **Hostname and Name resolution** parameters will help define the hostname (`Hostname`) and part of the FQDN (`Subdomain`) of the Pod, add hosts in the */etc/hosts* files of the containers (`HostAliases`), fine-tune the */etc/resolv.conf* files of the containers (`DNSConfig`) and define a policy for the DNS configuration (`DNSPolicy`).

- **Host namespaces** parameters will help indicate if the Pod must use host namespaces for network (`HostNetwork`), PIDs (`HostPID`), IPC (`HostIPC`) and if containers will share the same (non-host) process namespace (`ShareProcessNamespace`).

- **Service account** parameters will be useful to give specific rights to a Pod, by affecting it a specific service account (`ServiceAccountName`) or by disabling the automount of the default service account with `AutomountServiceAccountToken`.

- **Security Context** parameter (`SecurityContext`) help define various security attributes and common container settings at the pod level.

## Container Specs

The most important part of the definition of a Pod is the definition of the containers it will contain.

We can separate container parameters in two parts. The first part are parameters that are related to the container runtime (image, entrypoint, ports, environment variables and volumes), the second part are parameters that will be handled by the Kubernetes system.

The parameters related to container runtime are:

- **Image** parameters define the image of the container (`Image`) and the policy to pull the image (`ImagePullPolicy`).

- **Entrypoint** parameters define the command (`Command`) and its arguments (`Args`) of the entrypoint and its working directory (`WorkingDir`).

- **Ports** parameter (`Ports`) defines the list of ports to expose from the container.

- **Environment variables** parameters help define the environment variables that will be exported in the container, 
either directly (`Env`) or by referencing ConfigMap or Secret values (`EnvFrom`).

- **Volumes** parameters define the volumes to mount into the container, whether they are a filesystem volume (`VolumeMounts`) or a raw block volume (`VolumeDevices`).

The parameters related to Kubernetes are:

- **Resources** parameter (`Resources`) helps define the resource requirements and limits for a container.

- **Lifecycle** parameters help define handlers on lifecycle events (`Lifecycle`), parameterize the termination message (`TerminationMessagePath` and `TerminationMessagePolicy`), and define probes to check liveness (`LivenessProbe`) and readiness (`ReadinessProbe`) of the container.

- **Security Context** parameter help define various security attributes and common container settings at the container level.

- **Debugging** parameters are very specialized parameters, mostly for debugging purposes (`Stdin`, `StdinOnce` and `TTY`).

## Pod Controllers

The pod, although being the master piece of the Kubernetes architecture, is rarely used alone. You will generally use a Controller to run a pod with some specific policies.

The different controllers handling pods are:

- `ReplicaSet`: ensures that a specified number of pod replicas are running at any given time.
- `Deployment`: enables declarative updates for Pods and ReplicaSets.
- `StatefulSet`: manages updates of Pods and ReplicaSets taking care of stateful resources.
- `DaemonSet`: ensures that all or some nodes are running a copy of a Pod.
- `Job`: starts pods and ensures they complete.
- `CronJob`: creates a Job on a time-based schedule.

In Kubernetes, all controllers respect the principle of the **Reconcile Loop**: the controller perpetually **watches** for some objects of interest, to be able to detect if the actual state of the *world* (the objects running in the cluster) satisfies the specs of the different objects the controller is responsible for and to adapt the *world* consequently.

## ReplicaSet controller

Parameters for a ReplicaSet are:
- `Replicas` indicates how many replicas of selected Pods you want.
- `Selector` defines the Pods you want the ReplicaSet controller to manage.
- `Template` is the template used to create new Pods when insufficient replicas are detected by the Controller.
- `MinReadySeconds` indicates the number of seconds the controller should wait after a Pod starts without failing to consider the Pod is ready.

The ReplicaSet controller perpetually **watches** the Pods with the labels specified with `Selector`. At any given time, if the number of actual running Pods with these labels:
- is greater than the requested `Replicas`, some Pods will be terminated to satisfy the number of replicas. Note that the terminated Pods are not necessarily Pods that were created by the ReplicaSet controller.
- is lower than the requested `Replicas`, new Pods will be created with the specified Pod `Template` to satisfy the number of replicas. Note that to avoid the ReplicaSet controller to create Pods in a loop, the specified `Template` must create a Pod selectable by the specified `Selector`.

Note that:
- the `Selector` parameter of a ReplicaSet is immutable,
- changing the `Template` of a ReplicaSet will not have an immediate effect. It will affect the Pods that will be created after this change,
- changing the `Replicas` parameter will immediately trigger the creation or termination of Pods.

## Deployment controller

Parameters for a `Deployment` are:
- `Replicas` indicates the number of replicas requested.
- `Selector` defines the Pods you want the Deployment controller to manage.
- `Template` is the template requested for the Pods.
- `MinReadySeconds` indicates the number of seconds the controller should wait after a Pod starts without failing to consider the Pod is ready.
- `Strategy`
- `RevisionHistoryLimit`
- `Paused`
- `ProgressDeadlineSeconds`

The Deployment controller perpetually **watches** the ReplicaSets with the requested `Selector`. Among these:
- if a ReplicaSet with the requested `Template` exists, the controller will ensure that the number of `Replicas` for this ReplicaSet equals the number of requested `Replicas` (by using the requested `Strategy`) and `MinReadySeconds` equals the requested one.
- if no ReplicaSet exists with the requested `Template`, the controller will create a new ReplicaSet with the requested `Replicas`, `Selector`, `Template` and `MinReadySeconds`.
- for ReplicaSets with a non matching `Template`, the controller will ensure that the number of `Replicas` is set to zero (by using the requested `Strategy`).

Note that:
- the `Selector` parameter of a Deployment is immutable,
- changing the `Template` parameter of a Deployment will immediately:
  - either trigger the creation of a new ReplicaSet if no one exists with the requested `Selector` and `Template`
  - or update an existing one matching the requested `Selector` and `Template` with the requested `Replicas` (using `Strategy`),
  - set to zero the number of `Replicas` of other ReplicaSets,
- changing the `Replicas` or `MinReadySeconds` parameter of a Deployment will immediately update the corresponding value of the corresponding ReplicaSet (the one with the requested `Template`).
