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

## yaml validation

Of course there are many options, look at [learnk8s](https://learnk8s.io/validating-kubernetes-yaml) for a number of options.

For now lets go with kube-linter

```shell
wget https://github.com/stackrox/kube-linter/releases/download/v0.6.3/kube-linter-linux
```

Lets see what our linter says about our yaml

```shell
kube-linter lint 4.yaml
```

```shell
4.yaml: (object: <no namespace>/podinfo apps/v1, Kind=Deployment) container "podinfo" has cpu limit 0 (check: unset-cpu-requirements, remediation: Set CPU requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)
```

It complains on two things.

- no namespace
- no resource cpu limit set

I do not agree with any of these warnings.
Namespace can be set in a number of locations, like kustomize files.

And CPU limit creates throttling instead of restart like memory limits does. It's really hard to debug.
In general don't use CPU limits.

### Thoughts

I don't think a developer should have to add all of these configurations. In general I don't think that your users should need to know about half of them.

In general I think helm and kustomize becomes to complex, especially when you want to follow best practices.

If you think that all the settings should always be in your git repo then I think you should look in to [KRM framework](https://www.innoq.com/en/blog/kustomize-enhancement-with-krm-functions/
).

There is an excellent [talk](https://www.youtube.com/watch?v=rFtKguVs5jw&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D) around how Shopify manages there yaml by Katrina Verey. That I can recommend looking at.

Or why not use mutating webhooks to enforce best practices?

## OPA

Less yaml linting and more adminision webhooks and mutating webhooks.

Why not use PSP 2.0 or  Pod Security Admission [PSA](https://kubernetes.io/docs/concepts/security/pod-security-admission/) as it's calledd?

It's to limited and OPA or Kyverno solves more stuff and both projects are well backed.

Lets go with OPA for now, mostly because I know OPA more then Kyverno.

OPA have `constrainttemplate` and `constraint`, templates generates a CRD that you then can use to define which resources that should have specific constraints.
In constraints it's also where you can ignore specific deployments/namespaces like kube-system for example.
So we can add multiple constrainttemplates that in the end we don't use, it will "just" create extra CRDs.
There is also `asigns` that is the CRD that creates mutating webhooks, which actually applies the config that you enforce for you.
So you don't have to force your developers to learn super complex yaml. Instead let OPA/kyverno do everything for you.

```shell
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

[Gatekeeper library](https://github.com/open-policy-agent/gatekeeper-library) like the old PSP rule and others.

My old employer Xenit also had a number of good rules that we [open-sourced](https://github.com/XenitAB/gatekeeper-library/tree/main/library).

Lets revert to using our basic deployment.

```shell
kubectl apply -f 0.yaml
```

And lets apply some assign rules.
Notice that I use `-k` instead of `-f` for kustomize.

``` shell
kubectl apply -k 5
```

And for the next rollout we will now see the new setting being applied.

```shell
k rollout restart deployment/podinfo -n grafana
```

We will see that the pod got a seccompprofile attached to it self.
While the deployment don't got any mention of it.

## More OPA

So lets add some more stuff to our OPA config to reach the same config that we had earlier.
Lets use Xenits helm chart for this.

```shell
helm repo add gatekeeper-library https://xenitab.github.io/gatekeeper-library/
helm repo update
helm upgrade -i --version="0.23.1" gatekeeper-library-templates gatekeeper-library/gatekeeper-library-templates -n gatekeeper-system -f 6/values-templates.yaml
helm upgrade -i --version="0.23.1" gatekeeper-library-constraints gatekeeper-library/gatekeeper-library-constraints -n gatekeeper-system -f 6/values-constraint.yaml
```

Now lets see how it looks

```shell
k rollout restart deployment/podinfo -n grafana
```

In the future [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/
) will be able to remove a bunch of the constraint rules.
Sadly this is a alpha feature 1.26, so it's a long time until most people will use it.

## PDB and HPA

PDB is stupid

```shell
kubectl apply -f 7.yaml

```shell
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

## Trivy

CVE scanning of images helps you to get a start but it's not perfect.
Just like binaries you need to trust the source of container images that you download.

```shell
trivy image ghcr.io/stefanprodan/podinfo:6.3.6
# Scan not so great image
trivy image registry.k8s.io/hpa-example
```

There is a great talk from Kubecon 2023 that talks about how CVE scannings [works](https://www.youtube.com/watch?v=9weGi0csBZM&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D).

I really recommend looking at it.
But in short CVE scanning uses a bunch of meta data and if you delete these resources you will hide the CVEs from the scanners.

## NetworkPolicy

Use something like this as a default networkpolicy.

```shell
kubectl apply -f 8.yaml
```

If you are using [node-local-dns](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) you will need to open a few other ports.
I can really recommend using node-local-dns.

## service mesh

There are a many many service meshes.

- istio
- linkerd
- cilium
- istio ambient mesh
- etc.

In general my thinking is that you should only use it if you have compliant reasons to.
Even linkerd that is very simple to manage compare to istio is very painful.
The main reason to use something like istio is if you are doing a bunch of JWT stuff that auth and you want your mesh to help out with it.

If you only want/need mTLS look at cilium 1.13 or istio ambient mesh.

If you still want to play with service meshes, at least run Kubenretes in production for a year or 3 before trying it out.

## Prometheus metrics for security

During kubecon there was a talk called [Kubernetes Defensive Monitoring with Prometheus](https://www.youtube.com/watch?v=Ammf8oJhPkY&list=PLj6h78yzYM2PyrvCoOii4rAopBswfz1p7&index=120&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D),
they talk about what general k8s metrics you can use to detect anomalies.

It's a great talk and also a good way to learn how to write more advanced promeQl.
Here you can find the ppt from the [presentation](https://static.sched.com/hosted_files/kccnceu2023/8a/Defensive%20Monitoring%20KubeCon%20EU23%20David%20De%20Torres%20Mirco%20De%20Zorzi.pdf).
