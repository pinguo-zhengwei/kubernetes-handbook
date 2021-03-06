# Troubleshooting Pods

This chapter is about pods troubleshooting, which are applications deployed into Kubernetes.

Usually, no matter which errors are you run into, the first step is getting pod's current state and its logs

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

The pod events and its logs are usually helpful to identify the issue.

## Pod stuck in Pending

Pending state indicates the Pod hasn't been scheduled yet. Check pod events and they will show you why the pod is not scheduled. 

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x6 over 27s)  default-scheduler  0/4 nodes are available: 2 Insufficient cpu.
```

Generally this is because there are insufficient resources of one type or another that prevent scheduling. An incomplete list of things that could go wrong includes

- Cluster doesn't have enough resources, e.g. CPU, memory or GPU. You need to adjust pod's resource request or add new nodes to cluster
- Pod requests more resources than node's capacity. You need to adjust pod's resource request or add larger nodes with more resources to cluster
- Pod is using hostPort, but the port is already been taken by other services. Try using a Service if you're in such scenario

## Pod stuck in Waiting or ContainerCreating

In such case, Pod has been scheduled to a worker node, but it can't run on that machine.

Again, get information from `kubectl describe pod <pod-name>` and check what's wrong. An incomplete list of things that could go wrong includes

- Failed to pull image, e.g.
  - image name is wrong
  - registry is not accessible
  - image hasn't been pushed to registry
  - docker secret is wrong or not configured for secret image
  - timeout because of big size (adjusting kubelet  `--image-pull-progress-deadline` and `--runtime-request-timeout` could help for this case)
- Network setup error for pod's sandbox, e.g.
  - can't setup network for pod's netns because of CNI configure error
  - can't allocate IP address because exhausted podCIDR
- Failed to start container, e.g.
  - cmd or args configure error
  - image itself contains wrong binary

## Pod stuck in ImagePullBackOff

`ImagePullBackOff` means image can't be pulled by a few times of retries. It could be caused by wrong image name or incorrect docker secret. In such case, `docker pull <image>` could be used to verify whether the image is correct.

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason                 Age                From                                Message
  ----     ------                 ----               ----                                -------
  Normal   Scheduled              36s                default-scheduler                   Successfully assigned sh to k8s-agentpool1-38622806-0
  Normal   SuccessfulMountVolume  35s                kubelet, k8s-agentpool1-38622806-0  MountVolume.SetUp succeeded for volume "default-token-n4pn6"
  Normal   Pulling                17s (x2 over 33s)  kubelet, k8s-agentpool1-38622806-0  pulling image "a1pine"
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Failed to pull image "a1pine": rpc error: code = Unknown desc = Error response from daemon: repository a1pine not found: does not exist or no pull access
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Error: ErrImagePull
  Normal   SandboxChanged         4s (x7 over 28s)   kubelet, k8s-agentpool1-38622806-0  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff                4s (x5 over 25s)   kubelet, k8s-agentpool1-38622806-0  Back-off pulling image "a1pine"
  Warning  Failed                 1s (x6 over 25s)   kubelet, k8s-agentpool1-38622806-0  Error: ImagePullBackOff
```

For private images, a docker registry secret should be created

```sh
kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

and then refer the secret in container's spec:

```yaml
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: my-secret
```

## Pod stuck in CrashLoopBackOff

In such case, Pod has been started and then exited abnormally (its restartCount should be > 0). Take a look at the container logs

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

If your container has previously crashed, you can access the previous container’s crash log with:

```sh
kubectl logs --previous <pod-name>
```

From container logs, we may find the reason of crashing, e.g.

- Container process exited
- Health check failed
- OOMKilled

```sh
$ kubectl describe pod mypod
...
Containers:
  sh:
    Container ID:  docker://3f7a2ee0e7e0e16c22090a25f9b6e42b5c06ec049405bc34d3aa183060eb4906
    Image:         alpine
    Image ID:      docker-pullable://alpine@sha256:7b848083f93822dd21b0a2f14a110bd99f6efb4b838d499df6d04a49d0debf8b
    Port:          <none>
    Host Port:     <none>
    State:          Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     1
      memory:  1G
    Requests:
      cpu:        100m
      memory:     500M
...
```

Alternately, you can run commands inside that container with `exec`:

```sh
kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

If none of these approaches work, SSH to Pod's host and check kubelet or docker's logs. The host running the Pod could be found by running:

```sh
# Query Node
kubectl get pod <pod-name> -o wide

# SSH to Node
ssh <username>@<node-name>
```

## Pod stuck in Error

In such case, Pod has been scheduled but failed to start. Again, get information from `kubectl describe pod <pod-name>` and check what's wrong. Reasons include:

- referring non-exist ConfigMap, Secret or PV
- exceeding resource limits (e.g. LimitRange)
- violating PodSecurityPolicy
- not authorized to cluster resources (e.g. with RBAC enabled, rolebinding should be created for service account)

## Pod stuck in Terminating or Unknown

From v1.5, kube-controller-manager won't delete Pods because of Node unready. Instead, those Pods are marked with Terminating or Unknown status. If you are sure those Pods are not wanted any more, then there are three ways to delete them permanently

- Delete the node from cluster, e.g. `kubectl delete node <node-name>`. If you are running with a cloud provider, node should be removed automatically after the VM is deleted from cloud provider.
- Recover the node. After kubelet restarts, it will check Pods status with kube-apiserver and restarts or deletes those Pods.
- Force delete the Pods, e.g. `kubectl delete pods <pod> --grace-period=0 --force`. This way is not recommended, unless you know what you are doing. For Pods belonging to StatefulSet, deleting forcibly may result in data loss or split-brain problem.

## Pod is running but not doing what it should do

If the pod has been running but not behaving as you expected, there may be errors in your pod description. Often a section of the pod description is nested incorrectly, or a key name is typed incorrectly, and so the key is ignored.

Try to recreate the pod with `--validate` option:

```sh
kubectl delete pod mypod
kubectl create --validate -f mypod.yaml
```

or check whether created pod is expected by getting its description back:

```sh
kubectl get pod mypod -o yaml
```

## Static Pod not recreated after manifest changed

Kubelet monitors changes under `/etc/kubernetes/manifests`  (configured by kubelet's `--pod-manifest-path` option) directory by inotify. There is possible kubelet missed some events, which results in static Pod not recreated automatically. Restart kubelet should solve the problem.

## References

- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)