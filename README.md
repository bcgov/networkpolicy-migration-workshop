# Introduction

This guide will walk you through the migration from Aporeto Network Security Policy (NSP) to OpenShift / Kubernets Network Policy (KNP).

As you progress through the guide you will be able to leverage much of what you have done for NSP; for example, you can continue to use labels such as `role=web` or `component=api` that you may have created. The main difference bing now you'll be using `podSelector` and `namespaceSelector` syntax.

The current version of the OpenShift (v4.5) on the platform does not support all features outlined in the [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) documentation. The main differences as noted in the [OpenShift SDN](https://docs.openshift.com/container-platform/4.5/networking/openshift_sdn/about-openshift-sdn.html) documentation are that egress rules and some ipBlock rules are currently not supported; we expect these features to be delivered with OpenShift 4.8 later this fall.

 # Prologue

 Back in 2019 we decided to take a strong stance on security and, by way of a security focused project, began implementing several tools to make our OpenShift Container Platform (OCP) a leader in this respect. One of these tools, Aporeto, was chosen over KNP because it offered a way to extend security policies outside of OCP into other system. This would enable teams to write policy for OCP as well as external system.

 This went reasonably well on OCP4, however, significate issues have forced us to pivot to KNP. Some might say this was a failure, but in reality, learning new information and acting on it is a success. Learning new information and doing nothing would certainly be a failure.

# TL;DR

You're going to migrate from NSP to KNP by adding NSP to your namespace(s) to open up network communication so all pods can talk to all other pods within a given namespace. Then, you'll create new KNP to ratchet down communications to a more sane level.

* [OpenShift SDN](https://docs.openshift.com/container-platform/4.5/networking/openshift_sdn/about-openshift-sdn.html)

* [OpenShift NetworkPolicy](https://docs.openshift.com/container-platform/4.5/networking/network_policy/about-network-policy.html#about-network-policy)

* [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)


# Getting Started

If you already have NSP in place, and its working well, you can continue to use it until Aporeto is removed. When this happens, all NSP will be disabled, then deleted from the platform. It is recommended to apply the [QuickStart NSP](./quickstart-nsp.yaml) to your namespace to prevent Aporeto form enforcing network security in your namespace(s).

To do this, apply the [QuickStart NSP](./quickstart-nsp.yaml) with the following command:

```console
oc process -f quickstart-nsp.yaml -p NAMESPACE=$(oc project --short) | \
oc apply -f -
```

You'll find two new NSPs have been created:

```console
➜  nsp-to-knp-migration git:(master) ✗ oc get nsp
NAME              AGE
any-to-any        24s
any-to-external   24s
```

When you wrote NSP for your `Deployment` or `DeploymentConfig` you would have added a label to the pods so they could be identified by Aporeto; the recommended approach is to use something like:

* `role: web`
* `role: api`

    \- or - 

* `component: web`
* `component: api`


These same labels will be leveraged by your KNP to identify what pods KNP should be applied to. A typical deployment with the following components will only need three policies to be successfully:

* web
* api
* minio (object storage)
* database

The first policy will be to allow network traffic to enter your namespace (ingress); the second rule will be to allow your API to talk to the database; and the third rule will be to allow the API to talk to minio.

**Pro Tip 🤓**
- Once you add a policy to a pod any traffic not specifically permitted is rejected. You can leverage this behavior to simplify your policies;
- If you don't want a pod to accepted external network traffic (from the wild internet) then don't create a route to it. No additional policy is required.

## Ingress

Having a route alone isn't enough to let traffic flow into your pods, you also need a policy to specifically permit this. This will be the first policy we write. Once in place, any pod with an external route will receive traffic on said route.

```yaml
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-from-openshift-ingress
    labels:
      app: some-cool-app
  spec:
    ingress:
      - from:
          - namespaceSelector:
              matchLabels:
                network.openshift.io/policy-group: ingress
    podSelector: {}
    policyTypes:
      - Ingress
```

**Pro Tip 🤓**
- Add labels to your KNP to easily find and delete them as groups.
- `podSelector: {}` is a wildcard, if you want additional piece of mind add a label like `route-ingress: true` to pods that can accept external traffic and use it in place of the wildcard.

## Internal

Once external communication is enabled we need two additional policies to allow traffic between our pods. The fist policy, is to allow the API to talk to the database and the second is to allow the API to talk to minio. 

These examples use the label convention `component: api` but alternative like `role: api` are perfectly fine; patroni uses `role: master` to denote the primary replica so for this example `component` is better suited


```yaml
- kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: allow-api-to-patroni
    labels:
      app: some-cool-app
  spec:
    podSelector:
      matchLabels:
        cluster-name: patroni
    ingress:
      - from:
          - podSelector:
              matchLabels:
                component: api
        ports:
          - protocol: TCP
            port: 5432
- kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: allow-api-to-minio
    labels:
      app: some-cool-app
  spec:
    podSelector:
      matchLabels:
        component: minio
    ingress:
      - from:
          - podSelector:
              matchLabels:
                component: api
        ports:
          - protocol: TCP
            port: 9000
```

**Pro Tip 🤓**
- The `port` is whatever port the pod exposes.


## Testing

Test connectivity by opening a remote shell `oc rsh pod-name-here` to each pod then use the simple shell command show below:

```console
timeout 5 bash -c "</dev/tcp/api/8080"; echo $?
```

![How To Test](images/how-to-test.png)


| Item | Description |
| :--- | :---------- |
| A    | The protocol to use, `tcp` or `udp` |
| B    | The `service` name as shown by `oc get service` |
| C    | The port number exposed by the Pod |
| D    | The return code of the command: `0` means the pods can communicate, while `124` means the pods cannot communicate on the given protocol / port |
| E    | The delay in seconds the command will wait before failing |
