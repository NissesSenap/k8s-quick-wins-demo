# Demo

## 0

Create an example app

```shell
kubectl create deployment podinfo --image=ghcr.io/stefanprodan/podinfo:6.3.6 --dry-run=client -o yaml
```

Go through 0.yaml

## 1

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

## 2

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

### Capabilities

Ideally you will drop all capabilities, but due to reasons it might not be possible.

You need to drop

- NET_RAW
- SYS_ADMIN

### Seccomp profile

Seccomp (Secure Computing) is a feature in the Linux kernel that allows a userspace program to create syscall filters. In the context of containers, these syscall filters are collated into seccomp profiles that can be used to restrict which syscalls and arguments are permitted.

In k8s 1.27, this is enabled by default.