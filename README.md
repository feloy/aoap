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

The Pod created is ready for production ... it you are not very fussy. Otherwise, the Pod offers a long list of parameters to make it more production-ready. We will examine in this document all these parameters.

## Pod specs

Here is a classification of the Pod parameters in big categories.

- **Containers** parameters will define and parameterize more precisely each container of the Pod, whether it is a normal container (`Containers`) or an init container (`InitContainers`). The `ImagePullSecrets` parameter will help to download containers images from private registries.

- **Volumes** parameter (`Volumes`) will define a list of volumes that containers will be able to mount and share.

- **Scheduling** parameters will help you define the most appropriate Node to deploy the Pod, by selecting nodes labels (`NodeSelector`), directly specifying a node name (`NodeName`),  
using `Affinity` and `Tolerations`, by selecting a specific scheduler (`ScedulerName`), and by requiring a specific runtime class (`RuntimeClassName`). They will also be used to prioritize a Pod over other Pods (`PriorityClassName`and `Priority`).

- **Lifecycle** parameters will help define if a Pod should restart after execution ends (`RestartPolicy`) and fine-tune the periods after which processes running in the containers of a terminating pod are killed (`TerminationGracePeriodSeconds`) or after which a running Pod will be stopped if not yet terminated (`ActiveDeadlineSeconds`). They also help define readiness of a pod (`ReadinessGates`).

- **Hostname and Name resolution** parameters will help define the hostname (`Hostname`) and FQDN (`Subdomain`) of the Pod, add hosts in the `/etc/hosts` files of the containers (`HostAliases`), fine-tune the `/etc/resolv.conf` files of the containers (`DNSConfig`) and define a policy for the DNS configuration (`DNSPolicy`).

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
