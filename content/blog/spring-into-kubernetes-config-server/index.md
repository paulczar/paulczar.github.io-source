---
date: "2019-03-01"
title: "Spring into Kubernetes - Using Kubernetes as a Config Server"
categories: ["kubernetes","pivotal","spring"]
---

In previous installments of Spring into Kubernetes I've shown you how to [build images](https://tech.paulcz.net/blog/building-spring-docker-images/), [deploy applications](https://tech.paulcz.net/blog/spring-into-kubernetes-part-1/) and write a [Helm Chart](https://tech.paulcz.net/blog/spring-into-kubernetes-part-2/) for Spring applications. In this installment we'll look at [Spring Cloud Kubernetes](https://github.com/spring-cloud/spring-cloud-kubernetes) integrations, specifically using Kubernetes [Config Maps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files) as a Config Server.

Usually I would use a [Pivotal Container Service](https://pivotal.io/platform/pivotal-container-service) cluster to demonstrate, but in this demonstration I'll use a local [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) cluster.

## Spring Cloud Kubernetes

[Spring Cloud Kubernetes](https://github.com/spring-cloud/spring-cloud-kubernetes) brings in a ton of integrations with Kubernetes. This demonstration will focus just on the ability to integrate Kubernetes as a [configuration server](https://github.com/spring-cloud/spring-cloud-kubernetes#kubernetes-propertysource-implementations).

Minimal changes are needed to your applications, you need to simply add the following classes to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
```

You also need to enable it in your `bootstrap.yaml` (or `.properties`):

```yaml
spring.cloud.kubernetes.config.enabled: true
spring.cloud.kubernetes.reload.enabled: true
```

## Getting Started

You'll need to [install minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) by following the instructions provided for your Operating System. You'll also need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) so that you can give instructions to Kubernetes.

Start Minikube:

```console
$ minikube start
😄  minikube v0.34.1 on linux (amd64)
💡  Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
🔄  Restarting existing virtualbox VM for "minikube" ...
⌛  Waiting for SSH access ...
📶  "minikube" IP address is 192.168.99.103
🐳  Configuring Docker as the container runtime ...
✨  Preparing Kubernetes environment ...
🚜  Pulling images required by Kubernetes v1.13.3 ...
🔄  Relaunching Kubernetes v1.13.3 using kubeadm ...
⌛  Waiting for kube-proxy to come back up ...
🤔  Verifying component health ......
💗  kubectl is now configured to use "minikube"
🏄  Done! Thank you for using minikube!
```

Ensure that you can communicate with minikube:

```console
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   36h   v1.13.3
```

## Deploy Spring Hello World

The source for this demo can be found at [paulczar/spring-hello](https://github.com/paulczar/spring-helloworld) on github. Of importance it will respond to a web request with the contents of the application property `message`.

Deploy the example Hello World application and expose it via a service:

```console
$ kubectl run hello --image=paulczar/spring-hello:k8s001 --port=8080
deployment.apps/hello created

$ kubectl expose deployment hello --type=LoadBalancer --port 80 --target-port 8080
service/hello exposed
```

You can use `minikube service list` to get a list of services and the URLs for those services. This helps make up for the lack of LoadBalancer support in minikube:

```console
$ minikube service list
|-------------|------------|-----------------------------|
|  NAMESPACE  |    NAME    |             URL             |
|-------------|------------|-----------------------------|
| default     | hello      | http://192.168.99.103:30871 |
| default     | kubernetes | No node port                |
| kube-system | kube-dns   | No node port                |
|-------------|------------|-----------------------------|
```

Use the URL provided for the `hello` service:

```
$ curl http://192.168.99.103:30871
hello development
```

Our application is running and responding with `hello development`. This is the default value for `message` in the `development` spring profile.

## Configure Kubernetes support for Spring Hello World

If you have `rbac` enabled in your cluster (which you should, we're not animals) the service account your application is running and will be unable to view kubernetes resources. You can see these errors in the pod's logs:

```console
k logs deployment/hello
2019-03-01 16:13:02.629  WARN 1 --- [           main] o.s.cloud.kubernetes.StandardPodUtils    : Failed to get pod with name:[hello-bb9cf575d-rqt6n]. You should look into this if things aren't working as you expect. Are you missing serviceaccount permissions?

io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: GET at: https://10.96.0.1/api/v1/namespaces/default/pods/hello-bb9cf575d-rqt6n. Message: Forbidden!Configured service account doesn't have access. Service account may have been revoked. pods "hello-bb9cf575d-rqt6n" is forbidden: User "system:serviceaccount:default:default" cannot get resource "pods" in API group "" in the namespace "default".
...
...
```


You could create a new service user and give it the appropriate permissions, or you could give permissions to the default service account. Do the latter using a kubernetes manifest found [here](https://raw.githubusercontent.com/paulczar/spring-helloworld/master/deploy/rbac.yaml).


Update the `rolebinding` for the default user:

```console
$ kubectl apply -f https://raw.githubusercontent.com/paulczar/spring-helloworld/master/deploy/rbac.yaml
```

Delete the current pod to have the kubernetes deployment start a new one which should now have permissions:

```console
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hello-bb9cf575d-rqt6n   1/1     Running   0          12m

$ kubectl delete pod hello-bb9cf575d-rqt6n
pod "hello-bb9cf575d-rqt6n" deleted

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hello-bb9cf575d-8mwxq   1/1     Running   0          21s
```

If you look at the logs you'll see the error has disappeared and you can see it is looking for a `configmap`:

```console
$ kubectl logs deployment/hello
2019-03-01 16:26:33.080 DEBUG 1 --- [           main] o.s.cloud.kubernetes.config.ConfigUtils  : Config Map name has not been set, taking it from property/env spring.application.name (default=application)
2019-03-01 16:26:33.080 DEBUG 1 --- [           main] o.s.cloud.kubernetes.config.ConfigUtils  : Config Map namespace has not been set, taking it from client (ns=default)
2019-03-01 16:26:33.188  INFO 1 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource {name='composite-configmap', propertySources=[ConfigMapPropertySource@872306601 {name='configmap.hello.default', properties={}}]}

Since you haven't yet created a `configmap` the response from the application should still be the default:

```console
$ curl http://192.168.99.103:30871
hello development
```

Next create a `configmap` that the application will use, by default it will look for a `configmap` with the same name as the application:

```console
$ kubectl create configmap hello --from-literal=message="HELLO KUBERNETES"
configmap/hello created
```

Run the `curl` command again and you should see the new response:

```console
$ curl http://192.168.99.103:30871
HELLO KUBERNETES
```

## Conclusion

While this was a fairly simple demonstration of the Spring Cloud Kubernetes integrations you can see how useful it can be. By integrating directly into Kubernetes you can avoid running a [Spring Cloud Config](https://spring.io/projects/spring-cloud-config) service to get dynamic configuration of your application.

You can also load up an entire application properties file (either `.properties` or `.yaml`) inside a `configmap`, you can also store passwords and keys in a Kubernetes `secret` and dynamically load those.
