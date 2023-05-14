# Demo

## 0 basic deployment

Create an example app

```shell
kubectl create deployment podinfo --image=ghcr.io/stefanprodan/podinfo:6.3.6 --dry-run=client -o yaml
```

Go through 0.yaml

## 1 basic securityContext

Lets add spec.template.spec.securityContext.runAsNonRoot

```yaml
securityContext:
    runAsNonRoot: true
```

```shell
kubectl apply -f 1.yaml
```

Well that didn't go to well.
Sadly we will see an error that looks something like this.

```yaml
message: 'container has runAsNonRoot and image has non-numeric user (app),
    cannot verify user is non-root (pod: "podinfo-7456cc7897-hzbw9_grafana(9412359b-5c7f-474c-bdab-1d1545202c8e)",
```

Sadly we also need to add

```yaml
securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

The problem here is of course that we don't know the uid.

## 2 securityContext

Lets add a few other settings

```yaml
securityContext:
    allowPrivilegeEscalation: false
    capabilities: #
        drop:
        - ALL
    readOnlyRootFilesystem: true # read Only root filesystem
    runAsNonRoot: true # don't run as root
    runAsUser: 1000 # which user
    seccompProfile:
      type: RuntimeDefault #
```

```shell
kubectl apply -f 2.yaml
```

### Capabilities

Ideally you will drop all capabilities, but due to reasons it might not be possible.

You need to drop

- NET_RAW
- SYS_ADMIN

### Seccomp profile

Seccomp (Secure Computing) is a feature in the Linux kernel that allows a userspace program to create syscall filters. In the context of containers, these syscall filters are collated into seccomp profiles that can be used to restrict which syscalls and arguments are permitted.

In k8s 1.27, this is enabled by default.

## 3 Requests/limits

Request tells the scheduler where to put your pod. It's extermly important to set good requests if you want high usability of your nodes.

Requests is hard, but lets start to add something
Getting your developers to add something can be hard. Lets use LimitRange

```shell
kubectl apply -f 3-1.yaml
```

We will notice that nothing have changed. This will only hit on new pods. So lets restart podinfo.

```shell
k rollout restart deployment/podinfo
```

Lets overwrite the default

```yaml
resources:
    limits:
        memory: 512Mi
    requests:
        cpu: 100m
        memory: 64Mi
```

```shell
kubectl apply -f 3.yaml
```

To get help to generate correct requests use Vertical Pod Autoscaler [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler).
To generate VPA for all deployments use [goldilocks](https://www.fairwinds.com/goldilocks).

To make sure that all users set a default request help them with it.

## 4 disable service account token

99% of all deployments don't need service account tokens.

It can be disabled in both deployment and serviceAccount config.

`deployment.spec.template.spec.automountServiceAccountToken`

```yaml
automountServiceAccountToken: false
```
