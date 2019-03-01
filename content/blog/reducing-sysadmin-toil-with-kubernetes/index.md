---
date: "2019-01-29"
title: "Reducing Sysadmin toil with Kubernetes"
categories: ["kubernetes","pivotal","devops", "cloud-native"]
draft: true
---


# Resiliency all the way down

There is a fundamental aspect of Kubernetes that will be a force multiplier for making composable building blocks for operations teams managing cloud native environments. It starts with the Kubernetes controller.

Most resources in Kubernetes are managed by kube-controller-manager, or a controller for short. A controller is defined as "a control loop that watches the shared state of a cluster ... and makes changes attempting to move the current state toward the desired state" (1). Think of it like this: a Kubernetes controller is to a microservice as a Chef recipe (or Ansible playbook) is to a monolith.

Each Kubernetes resource is controlled by its own control loop. This is a step forward from previous systems like Chef or Puppet which both have control loops but at the server level, not the resource level. A Controller is a fairly simple piece of code that creates a control loop over a single resource to ensure that resource is behaving correctly. These control loops can stack together to create complex functionality with simple interfaces.

The canonical example of this in action is in how we manage pods in Kubernetes. A pod is effectively a running copy of your application that a specific worker node is asked to run. If that application crashes the kubelet running on that node will start it again. However, if that node crashes the pod is not recovered, as the control loop (via the kubelet process) responsible for the resource no longer exists. To make applications more resilient, Kubernetes has the ReplicaSet controller.

The Replicaset controller is bundled inside the Kubernetes `controller-manager` which runs on the Kubernetes master node and contains the controllers for these more advanced resources. The ReplicaSet controller is responsible for ensuring that a set number of copies of your application are always running. To do this, the ReplicaSet controller requests that a given number of pods are created. It then routinely checks that the correct number of Pods are still running, and will request more pods or destroy existing pods to do so.

By requesting a ReplicaSet from Kubernetes you get a self-healing deployment of your application. You can further add lifecycle management to your workload by requesting [a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), which is a controller that manages ReplicaSets and provides rolling upgrades by managing ReplicaSets of multiple versions of your application.

These Controllers are great for managing Kubernetes resources, but are also fantastic for managing resources outside of Kubernetes. The [Cloud Controller Manager](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/) is a grouping of Kubernetes Controllers that act on resources external to Kubernetes, specifically resources that provide functionality to Kubernetes on the underlying cloud infrastructure. This is what drives Kubernetes ability to do things like a [Service](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) of type `LoadBalancer` and have it create and manage a cloud specific loadbalancer such as an ElasticLoadBalancer on AWS.

Furthermore you can extend Kubernetes by writing a Controller that watches for events and annotations and performs extra work, either acting on Kubernetes resources, or external resources that have some form of programmable API.

To review:

* Controllers are a fundamental building block of Kubernetes' functionality.
* A controller forms a control loop to ensure the state of a given resource matches the requested state.
* Kubernetes provides controllers via __Controller Manager__ and __Cloud Controller Manager__ processes that provide additional resilience and functionality.
* The ReplicaSet controller adds resiliency to pods by ensuring the correct number of replicas are running.
* A Deployment controller adds rolling upgrade capabilities to ReplicaSets.
* You can extend Kubernetes functionality by writing your own Controllers.

# Controllers reduce Sysadmin Toil

Some of the most frequent tickets that come into a Sysadmin ticket queue are for performing fairly simple tasks that should be easily automated, but for various reasons are not. For example creating or updating a DNS record generally requires updating a [zone](https://en.wikipedia.org/wiki/Zone_file) file, one bad entry and you could take down your entire DNS infrastructure. Or how about those tickets that look like __[SYSAD-42214] Expired SSL Certificate - Production is down__.

What if I told you that Kubernetes could manage these things for you by running some additional Controllers. Imagine a world where by asking Kubernetes to run applications for you it automatically created and managed both DNS addresses and SSL certificates. What a world we live in!

## Example - External DNS Controller

The [external-dns](https://github.com/kubernetes-incubator/external-dns) controller is a perfect example of Kubernetes treating operations as a microservice. You configure it with your DNS provider and it will watch resources such as Services and Ingress Controllers. When one of those resources changes it will inspect them for annotations which will tell it if it needs to perform an action.

With the `external-dns` controller running in your cluster you can simply add the following annotation to a service and it will go out and create a matching DNS A record for that resource:

```console
kubectl annotate service nginx \
    "external-dns.alpha.kubernetes.io/hostname=nginx.example.org."
```

You can change other characteristics such as the TTL value of the DNS record:

```console
kubectl annotate service nginx \
    "external-dns.alpha.kubernetes.io/ttl=10"
```

Just like that you now have automatic DNS management for your applications and services in Kubernetes that reacts to any changes in your cluster to ensure your DNS is correct.

## Example - Certificate Manager Operator

Like the `external-dns` controller the [cert-manager](http://docs.cert-manager.io/en/latest/) will react to changes in resources, but also comes with a Custom Resource Definition that will allow you to request certificates as a resource in of themselves, not just a byproduct of an annotation.

`cert-manager` works with [Lets Encrypt](https://letsencrypt.org/) and other sources of Certificates to request valid signed TLS certificates. You can even use it in combination with `external-dns` like the following which will register `web.example.com` and retrieve a TLS certificate from Lets Encrypt and store that in a Secret.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    certmanager.k8s.io/acme-http01-edit-in-place: "true"
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
  name: example
spec:
  rules:
  - host: web.example.com
    http:
      paths:
      - backend:
          serviceName: example
          servicePort: 80
        path: /*
  tls:
  - hosts:
    - web.example.com
    secretName: example-tls
```

You can also request a certificate directly from the `cert-manager` CRD like so which like above will result in a certificate keypair being stored in a Kubernetes secret:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-staging
  commonName: example.com
  dnsNames:
  - www.example.com
  acme:
    config:
    - http01:
        ingressClass: nginx
      domains:
      - example.com
    - http01:
        ingress: my-ingress
      domains:
      - www.example.com
```

# Conclusion

This was just a quick look at one of the ways that Kubernetes is helping enable a new wave of changes to how we operate software. This is a favorite topic of mine, I look forward to sharing more on opensource.com and my personal blog. I'd like to hear how you use controllers as well - message me on Twitter [@pczarkowski](https://twitter.com/pczarkowski).

(1) https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/