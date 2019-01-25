---
date: "2019-01-29"
title: "Cloud Native Operations - Kubernetes Controllers"
categories: ["kubernetes","pivotal","devops", "cloud-native"]
---

# Lulz what? Cloud Native Operations ?!?!?!

Historically Operations practices have lagged behind development. During the 90s a number of lightweight software development practices evolved such as Scrum and Extreme Programming. During the early 2000's it became pretty common to practice (or at least claim to) some form of Agile in software development.

It wasn't until the last year of that decade that we started to see an uptick in Operations folks wanting to adopt Agile type methodologies and as the devops (and later SRE) movements took off we started to borrow heavily from Lean principals such as Kanban and Value Stream Mapping.

Cloud Computing has brought about another shift in software development, going from large monolithic applications to collections of microservices that work together, and even further into being event based via messages and streams which now falls under the umbrella of "cloud native".

With the rise of Kubernetes and similar platforms as well as companies like Hashicorp and stalwarts of Agile Operations such as Google and Pivotal we're starting to see that same shift in Operations as we start to talk about __Platform as Product__ and turning engineering [operations] teams into product teams.

{{< tweet 1088673759291011073 >}}

# Kubernetes Controllers

There's a lot more to be said about Cloud Native Operations and Platform as Product (the two go hand-in-hand) but for now I want to focus on a fundamental aspect of Kubernetes that will be a force multiplier for making the composable building blocks of Cloud Native Operations.

Most resources in Kubernetes are managed by a Controller. A Kubernetes Controller is to microservices what a Chef recipe is to a Monolith.

Each resource is controlled by its own control loop. This is a step forward from previous systems like Chef or Puppet which both have control loops but at the server level, not the resource.

A Controller is a fairly simple piece of code that creates a control loop over a single resource to ensure that resource is behaving correctly. These Control loops cam stack together to create complex functionality with simple interfaces.

The canonical example of this in action is in how we manage pods in Kubernetes. A Pod is [effectively] a running copy of your application that a specific worker node is asked to run. If that application crashes the kubelet running on that node will start it again.

However if that node crashes the Pod is not recovered as the control loop (via the kubelet process) responsible for the resource no longer exists. To make applications more resiliant Kubernetes has the ReplicaSet controller.

Kubernetes has a process running on the masters called a `controller-manager` that run the controllers for these more advanced resources. This is where the ReplicaSet controller runs, and it is responsible for ensuring that a set number of copies of your application are always running.

To do this the ReplicaSet controller requests that the provided number of Pods are created and then it routinely checks that the correct number of Pods are still running and will request more pods, or destroy existing pods to do so.

By requesting a ReplicaSet from Kubernetes you get a self-healing deployment of your application. You can further add lifecycle management to your workload by requesting a Deployment which is a controller that manages ReplicaSets.

These Controllers are great for managing Kubernetes resources, but are also fantastic for managing resources outside of Kubernetes. You can extend Kubernetes by writing a Controller that watches for events and annotations and performs extra work, or by writing a Custom Resource Definition.

# Example - External DNS Controller

The [external-dns](https://github.com/kubernetes-incubator/external-dns) controller is a perfect example of a watcher. You configure it with your DNS provider and it will watch resources such as Services and Ingresses. When one of those resources changes it will inspect them for annotations which will tell it if it needs to perform an action.

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

# Example - Certificate Manager Operator

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

This was just a quick look at one of the ways that Kubernetes is helping enable a new wave of changes to how we operate software. This is a favorite topic of mine, so look forward to hearing more.
