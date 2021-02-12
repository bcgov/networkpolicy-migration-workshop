# TL;DR

You're going to migrate from Aporeto Network Security Policy (NSP) to OpenShift's implementation of Kubernetes Network Policy (KNP) by first adding NSP to your namespace(s) to open up network communication so all pods can talk to all other pods within a given namespace. Then, you'll create new KNP to ratchet down communications to a sane level.

Here are some additional documents that thoroughly explain OCP's SDN implementation of KNP.

* [OpenShift SDN](https://docs.openshift.com/container-platform/4.5/networking/openshift_sdn/about-openshift-sdn.html)

* [OpenShift NetworkPolicy](https://docs.openshift.com/container-platform/4.5/networking/network_policy/about-network-policy.html#about-network-policy)

* [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

# Prologue

 Back in 2019 we decided to take a strong stance on security and, by way of a security focused project, began implementing several tools to make our OpenShift Container Platform (OCP) a leader in this respect. One of these tools, Aporeto, was chosen as a Software Defined Network solution to control network security for Platform app. Aporeto has been selected over Openshift 4 Built-In SDN capability powered by Kubernetes Network Policy (KNP), because it offered a way to extend security policies outside of Openshift into other systems that are based on the traditional infrastructure such as databases hosted in Zone B. This would have enabled teams to secure connections between their apps running in the Openshift Platform and datasources hosted inside the Zone B network zone. 

While Aporeto provided a great developer experience and the functionality that met our needs very well, we ran into some issues with running it on top of our specific Openshift implementation and thus, the decision to pivot to OCP 4 Built-In SDN. Some might say this was a failure, but in reality, learning new information and acting on it is a success. Learning new information and doing nothing would certainly be a failure.

**Takeaway 🧐**
- Aporeto and Kubernetes NetworkPolicy have a fairly comparable impact from the end-user’s (platform tenant) perspective. The main difference is that Aporeto could be extended to external systems where as KNP only applies to OCP. We are actively looking into the workarounds for the teams that need to secure integrations between their OpenShift applications and Zone B components and expect to finalize the list of options in April 2021.

# Introduction

This guide will walk you through the migration from Aporeto Network Security Policy (NSP) to OpenShift / Kubernetes Network Policy.

As you progress through the guide you will be able to leverage much of what you have done for NSP; for example, you can continue to use labels such as `role=web` or `component=api` that you may have created. 

The current version of the OpenShift (v4.5) on the platform does not support all features outlined in the [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) documentation. The main differences as noted in the [OpenShift SDN](https://docs.openshift.com/container-platform/4.5/networking/openshift_sdn/about-openshift-sdn.html) documentation are that egress rules and some ipBlock rules are currently not supported; we expect these features to be delivered with OpenShift 4.8 later this fall.

If you need egress rules to limit what your pods can communicate with contact Platform Services (PS) in #devops-aporeto Rocketchat channel. We can help implement this type of policy.

# Getting Started

Before we dive into migrating you to KNP lets go over a few important details:

### Egress Rules

With KNP in place pods will be able to connect to other pods within their namespace, in other namespaces, or to external systems (outside of the cluster). This is because egress rules are not available to tenants (project teams) just yet. This type of policy is available in OCP v4.6 but there isn't a migration path to them until OCP v4.8 which is expected in June of this year (2021). 

Without egress policy, pods that need to communicate **between namespaces** only require ingress rules on the destination pod to permit the inbound communication. This is fine is most circumstances because you will have a *deny-by-default* rule guarding all your namespaces.

High security projects that require egress rules to isolated a namespace should reach out to Platform Services; these policies can be implemented, as needed, by a cluster administrator.

### Roll Out

As platform tenants implement network policy they are "rolling out" KNP; there is nothing Platform Services needs to do. Everything is in place and working as expected. What will happen as per the dates below is a `cut-over` process were a *deny-by-default* policy will be installed in each namespace, if it does not already exist. This policy can not be removed or altered. Any existing KNP or NSP will not be touched.

Once the *deny-by-default* policy is in place it will mirror how OCP4 namespaces were delivered: locked down, no pods can talk within a namespace, no pods can talk across namespaces. Teams will need to build up KNP to permit pods to communicate as they see fit, with the exception of egress as per above.

If you already have NSP in place, and it's working, you can continue to use it until Aporeto is removed. When this happens, all NSP will be disabled, then after a soak-in period it will be deleted and Aporeto will be un-installed from the platform.

### Schedule 

Here is the schedule of events. All changes take places during business hours on a Monday (Read-Only Friday is in effect).

| Date      | What Happens? |
| :-------- | :------------ |
| March 1   | All `tools` and `dev` namespaces are cut-over to KNP. |
| March 8   | All `test` namespaces are cut-over to KNP. |
| March 15  | All `prod` namespaces are cut-over to KNP. |
| March 29  | Aporeto will be disable but still be installed on the cluster. |
| April 5   | Aporeto will be un-installed from the cluster. |


# Implementation

When KNP is added to a namespace that targets a pod, that's when the control traffic flow at the IP address or port level (OSI layer 3 or 4) takes effect. This is why **the cut-over dates above are important**: The *deny-by-default* policy automatically targets all pods within a namespace and effectively "turns on" KNP. With this rule in place teams won't be impacted by the cut-over activities.

## Quick Start

There is an OCP template called [QuickStart](./quickstart.yaml) at the root level of this repo. It adds very permissive NSP effectively disabling it for the given namespace while simultaneously adding KNP to lock down the namespace (with the *deny-by-default* policy), then opening up ingress (inbound communication) for any pod with a service / route combination, and finally allowing all pods within the namespace to communicate.

This will essentially migrate a namespace from NSP to KNP in such a way that the cut-over activities (schedule above) will have no impact on a namespace.

Before you run the quick start template, consider removing any excess NSP so that debugging is easier:

```console
oc delete nsp,en --all
```

When you are ready to apply the quick start policy above run the following command passing in the two required parameters described below:

```console
oc process -f quickstart.yaml \
 -p NAMESPACE_PREFIX=<LICENS_PLATE_HERE> \
 -p ENVIRONMENT=<ENVIRONMENT_NAME_HERE> | \
 oc apply -f -
```

| Parameter        | Description         |
| :--------------- | :------------------ |
| NAMESPACE_PREFIX | The license plate portion of your namespace. ie a 96fec |
| ENVIRONMENT      | The name of your environment. ie. dev |

Here is what the command should look like when run:

```console
➜  knp-migration-workshop git:(main) ✗ oc process -f quickstart.yaml NAMESPACE_PREFIX=${$(oc project --short)%%"-"*} -p ENVIRONMENT=${$(oc project --short)##*"-"} | oc apply -f -
networkpolicy.networking.k8s.io/deny-by-default created
networkpolicy.networking.k8s.io/allow-all-internal created
networksecuritypolicy.security.devops.gov.bc.ca/any-to-any created
networksecuritypolicy.security.devops.gov.bc.ca/any-to-external created
```

That's it. While you're technically done it is **highly** recommended teams write custom policy that deliberately controls how pods communicate. To learn more about this keep reading...

**Pro Tip 🤓**

- Use `oc get networkpolicy` or the OpenShift Web Console to view your newly minted policy;
- You can use this command to run the policy against your current namespace:

```console
oc process -f quickstart.yaml \
 -p NAMESPACE_PREFIX=${$(oc project --short)%%"-"*} \
 -p ENVIRONMENT=${$(oc project --short)##*"-"} | \
 oc apply -f -
```

## Migrating Custom Network Policy

Writing custom network policy to control traffic flow between pods is by far the better approach to securing a namespace. The the docs linked in the **TL;DR** section above detail how to write policy. This guide is intended to be a high-level overview to better understand and convert existing NSP to KNP.

### Side by Side Comparison

The left panel of the image below shows existing NSP while the right shows it converted to KNP.

![Side by Side](images/side-by-side.png)

Let's go through the differences in anatomy of NSP and KNP to see how they translate:

![Side by Side 2](images/side-by-side-2.png)

1) The **red** box shows the "header" of the YAML. It works just like any other OCP/K8S object definition. The main differences being the `kind` and `apiVersion` have changed.

2) The **yellow** box illustrates the description. In NPS this was programmed into the Aporeto Web Control plane so one could better understand what the policy was meant to do. This isn't applicable in KNP, however, it's **highly recommended** you documents **why** the policy is needed. Those who follow in your footsteps may not have the clear understanding of communication paths as you do.

3) The **green** box illustrates the source component; this is where the communication originates. The NSP on the left uses label to identify pods as does the KNP on the right. The only difference being KNP uses the `podSelector` notation.

    In Aporeto NSP you would have added a `$namespace` label to specify where the pod was expected to live. In KNP the *deny-by-default* policy walls off your namespace; all policy builds on top of this and it is assumed pods live in the same namespace. If you are doing cross-namespace communication you'll need to add a `namespaceSelector` to the `from` section of the YAML.

    **Pro Tip 🤓**
    - Policies are additive, meaning they build on one another like a logical `AND`;
    - Use `oc describe namespace/abc123-dev` to see what labels you can use to uniquely identify a namespace. You cannot use fields like `name: abc123-dev`; only labels work.

3) Finally, the **purple** box illustrates the destination component; this is where the communication ends. Again, it uses a `podSelector` to match labels on the destination pod and assumes pods are in the same namespace thanks to the *deny-by-default* policy. There is no need to add a `namespaceSelector` selector here as the policy is in the same namespace as the pod.

### Labeling Your Pods

When you wrote NSP for your `Deployment` or `DeploymentConfig` you would have added a label to the pods so they could be identified by Aporeto. For example, you may have used labels like this in the template section of a `DeploymentConfig`:

```yaml
      template:
        metadata:
          name: web
          labels:
            app: mobile-signing-service
            component: web
        spec:
```

An excellent alternative to `component` is to use `role` like this example:

```yaml
      template:
        metadata:
          name: api
          labels:
            app: mobile-signing-service
            role: api
        spec:
```

These labels will be leveraged by your KNP to identify what pods KNP should be applied to.

### Example

Let's take a look at writing (converting) KNP for a simple example with the components listed below; each component below will have its own `DeploymentConfig`. For this example only four policies are needed: The first is the *deny-by-default* referred to above. The second policy will be to allow network traffic to enter your namespace (ingress); the third policy will be to allow the API to talk to the database; and the fourth policy will be to allow the API to talk to minio (S3 compatible object store).

**Components**
* web
* api
* minio (object storage)
* database


**Pro Tip 🤓**
- Once you add a policy to a pod any traffic not specifically permitted is rejected. You can leverage this behavior to simplify your policies.
- If you don't want a pod to accepted external network traffic (from the wild internet) then don't create a route to it. No additional policy is required.
- `patroni` uses the label `role=master` to identify the primary replica in a `StatefulSet`, this examples uses the label `component` to identify pods to avoid confusion with this label.
- Did you know CITZ offers an alternative solution to using minio? Learn more about it [here](https://github.com/BCDevOps/OpenShift4-Migration/issues/59)

**Walled Garden**

First we'll isolate the namespace creating a walled garden. Nothing will be able to talk to the pods inside and the pods inside won't be able to talk to one another:

```yaml
- kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: deny-by-default
  spec:
    # The default posture for a security first namespace is to
    # deny all traffic. If not added this rule will be added
    # by Platform Services during environment cut-over.
    podSelector: {}
    ingress: []
```

**Ingress**

Having a route alone isn't enough to let traffic flow into your pods, you also need a policy to specifically allow this. This will be the second policy written. Once in place, any pod with an external route will receive traffic on said route.

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
- Add labels to your KNP to easily find and delete them as a group.
- `podSelector: {}` is a wildcard, if you want additional piece of mind add a label like `route-ingress: true` to pods that can accept external traffic and use it in place of the wildcard.

**Pod to Pod**

Once external communication is enabled we need two additional policies to allow traffic between pods. The fist policy below allows the API to talk to the database and the second allows the API to talk to minio. 

These examples use the label convention `component: api`, but alternatives like `role: api` are perfectly fine; patroni uses `role: master` to denote the primary replica so for this example `component` is better suited

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
- The `port` is whatever port the pod exposes. It has nothing to do with the `Service`.


## Testing

Test connectivity by opening a remote shell `oc rsh pod-name-here` to each pod then use the simple shell command shown below:

```console
timeout 5 bash -c "</dev/tcp/api/8080"; echo $?
```

![How To Test](images/how-to-test.png)


| Item | Description |
| :--- | :---------- |
| A    | The protocol to use, `tcp` or `udp` |
| B    | The `service` or pod name as shown by `oc get service` or `oc get pods` |
| C    | The port number exposed by the Pod |
| D    | The return code of the command: `0` means the pods can communicate, while `124` means the pods cannot communicate on the given protocol / port |
| E    | The delay in seconds the command will wait before failing |

## Need More Help?

If you need more help after reading this please ask questions in the `#devops-how-to` or do a quick search through [these issues](https://github.com/BCDevOps/OpenShift4-Migration/issues) in our OCP4 migration Q&A repo.
