+++
title = "Writing Your First Helm Chart"
description = ""
categories = ["kubernetes", "helm", "k8s"]
date = "2017-10-03"
draft = false
+++

I recently found myself writing [instructions](https://github.com/IBM/activator-lagom-java-chirper/blob/master/docs/README.md) on how to deploy an
application to several Kubernetes platform and ended up writing a different Kubernetes manifests for each
platform. 95% of the content was the same with just a few different directives
based on how the particular platform handles ingress, or if we needed a Registry secret or a TLS certificate.

Kubernetes manifests are very declarative and don't offer any way to put conditionals or variables that could be set in them. This
is both a good and a bad thing. Enter [Helm](https://docs.helm.sh/) a Package Manager for Kubernetes. Helm allows you to package up
your Kubernetes application as a package that can be deployed easily to Kubernetes, One of its features (and the one that interested me)
the ability to template out your Kubernetes manifests.

If you already have a Kubernetes manifest its very easy to turn it into a Helm Chart that you can then
iterate over and improve as you need to add more flexibility to it. In fact your first iteration of a
Helm chart can be as simple as moving your manifests into a new directory and adding a few lines
to a Chart.yaml file.

## Prerequisites

You'll need the following installed to follow along with this tutorial:

* [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
* [Helm](https://github.com/kubernetes/helm/blob/master/docs/install.md)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Prepare Environment

Bring up a test Kubernetes environment using Minikube:

```bash
$ minikube start
Starting local Kubernetes v1.7.5 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
```

Wait a minute or so and install Helm's tiller service to Kubernetes:

```bash
$ $HELM_HOME has been configured at /home/pczarkowski/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Happy Helming!
```

Create a path to work in:

```bash
$ mkdir -p ~/development/my-first-helm-chart
$ cd ~/development/my-first-helm-chart
```

_If it fails out you may need to wait a few more minutes for minikube to become
accessible._

## Create Example Kubernetes Manifest.

Writing a Helm Chart is easier when you're starting with an existing set of
Kubernetes manifests. One of the easiest ways to get a basic working manifest
is to ask Kubernetes to
[run something and then fetch the manifest](https://blog.heptio.com/using-kubectl-to-jumpstart-a-yaml-file-heptioprotip-6f5b8a63a3ea).

```bash
$ mkdir manifests
$ kubectl run example --image=nginx:1.13.5-alpine \
    -o yaml > manifests/deployment.yaml
$ kubectl expose deployment example --port=80 --type=NodePort \
    -o yaml > manifests/service.yaml
$ minikube service example --url       
http://192.168.99.100:30254
```

All going well you should be able to hit the provided URL and get the "Welcome to nginx!"
page. You'll see you now have two Kubernetes manifests saved. We can use these
to bootstrap our helm charts:

```bash
$ tree manifests
manifests
├── deployment.yaml
└── service.yaml
0 directories, 2 files
```

Before we move on we should clean up our environment.  We can use the newly
created manifests to help:

```bash
$ kubectl delete -f manifests
deployment "example" deleted
service "example" deleted
```

## Create and Deploy a Basic Helm Chart

Helm has some tooling to create the scaffolding needed to start developing a
new Helm Chart. We'll create it with a placeholder name of `helm`:

```bash
$ helm create helm
Creating helm
tree helm
helm
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
2 directories, 7 files
```

Helm will have created a number of files and directories.

* `Chart.yaml` - the metadata for your Helm Chart.
* `values.yaml` - values that can be used as variables in your templates.
* `templates/*.yaml` - Example Kubernetes manifests.
* `_helpers.tpl` - helper functions that can be used inside the templates.
* `templates/NOTES.txt` - templated notes that are displayed on Chart install.

Edit `Chart.yaml` so that it looks like this:

```yaml
apiVersion: v1
description: My First Helm Chart - NGINX Example
name: my-first-helm-chart
version: 0.1.0
```

Copy our example Kubernetes manifests over the provided templates and remove the
currently unused `ingress.yaml` and `NOTES.txt`.

```bash
$ cp manifests/* helm/templates/
$ rm helm/templates/ingress.yaml
$ rm helm/templates/NOTES.txt
```

Next we should be able to install our helm chart which will deploy our application
to Kubernetes:

```bash
$ helm install -n my-first-helm-chart helm
NAME:   my-first-helm-chart
LAST DEPLOYED: Tue Oct  3 10:20:57 2017
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME     CLUSTER-IP  EXTERNAL-IP  PORT(S)       AGE
example  10.0.0.210  <nodes>      80:30254/TCP  0s

==> v1beta1/Deployment
NAME     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
example  1        1        1           0          0s

```
_Like before we can use `minikube` to get the URL:_

```bash
$ minikube service example --url    
http://192.168.99.100:30254
```
Again accessing that URL via your we browser should get you the default NGINX welcome page.

Congratulations!  You've just created and deployed your first Helm chart. However we're
not quite done yet. use Helm to delete your deployment and then lets move on to customizing
the Helm Chart with variables and values:

```bash
$ helm del --purge my-first-helm-chart
release "my-first-helm-chart" deleted
```

## Add variables to your Helm Chart

Check out `helm/values.yaml` and you'll see there's a lot of variables already
defined by the example that helm provided when you created the helm chart. You'll
notice that it is has values for `nginx` in there. This is because Helm also uses
nginx as their example. We can re-use some of the values provided but we should clean
it up a bit.

Edit `helm/values.yaml` to look like this:

```yaml
replicaCount: 1
image:
  repository: nginx
  tag: 1.13.5-alpine
  pullPolicy: IfNotPresent
  pullSecret:
service:
  type: NodePort
```

We can access any of these values in our templates using the golang templating
engine. For example accessing `replicaCount` would be written as `{{ .Values.replicaCount }}`.
Helm also provides information about the Chart and Release which we'll also utilize.

Update your `helm/templates/deployment.yaml` to utilize our values:

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  creationTimestamp: 2017-10-03T15:03:17Z
  generation: 1
  labels:
    run: "{{ .Release.Name }}"
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"     
  name: "{{ .Release.Name }}"
  namespace: default
  resourceVersion: "3030"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/example
  uid: fd03ac95-a84b-11e7-a417-0800277e13b3
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      run: "{{ .Release.Name }}"
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: "{{ .Release.Name }}"
    spec:
      {{- if .Values.image.pullSecret }}    
            imagePullSecrets:
              - name: "{{ .Values.image.pullSecret }}"
      {{- end }}          
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        name: example
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
```
_Note the use of the `if` statement around `image.pullSecret` being set. This
sort of conditional becomes very important when making your Helm Chart portable across
different Kubernetes platforms._

Next edit your `helm/templates/service.yaml` to look like:

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2017-10-03T15:03:30Z
  labels:
    run: "{{ .Release.Name }}"
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"  
  name: "{{ .Release.Name }}"
  namespace: default
  resourceVersion: "3066"
  selfLink: /api/v1/namespaces/default/services/example
  uid: 044d2b7e-a84c-11e7-a417-0800277e13b3
spec:
  clusterIP:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30254
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: "{{ .Release.Name }}"
  sessionAffinity: None
  type: "{{ .Values.service.type }}"
status:
  loadBalancer: {}
```

Once your files are written out you should be able to install the Helm Chart:

```bash
$ helm install -n second helm
NAME:   second
LAST DEPLOYED: Tue Oct  3 10:59:41 2017
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME    CLUSTER-IP  EXTERNAL-IP  PORT(S)       AGE
second  10.0.0.160  <nodes>      80:30254/TCP  1s

==> v1beta1/Deployment
NAME    DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
second  1        1        1           0          1s
```

Next use minikube to get the URL of the service, but since we templated the
service name to match the release you'll want to use this new name:

```bash
$ minikube service second --url
http://192.168.99.100:30254
```

Now lets try something fun. Change the image we're using by upgrading the helm release
and overriding some values on the command line:

```
$ helm upgrade --set image.repository=httpd --set image.tag=2.2.34-alpine second helm
Release "second" has been upgraded. Happy Helming!
LAST DEPLOYED: Tue Oct  3 11:09:30 2017
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME    CLUSTER-IP  EXTERNAL-IP  PORT(S)       AGE
second  10.0.0.160  <nodes>      80:30254/TCP  9m

==> v1beta1/Deployment
NAME    DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
second  1        1        1           0          9m
```

Then go to our minikube provided URL and you'll see a different message `It works!`.

## Clean up

use `minikube delete` to clean up your environment:

```
minikube delete
Deleting local Kubernetes cluster...
Machine deleted.
```

## Conclusion

Helm is a very powerful way to package up your Kubernetes manifests to make them
extensible and portable. While it is quite complicated its fairly easy to get started
with it and if you're like me you'll find yourself replacing the Kubernetes manifests
in your code repos with Helm Charts.

There's a lot more you can do with Helm, we've just scratched the surface. Enjoy
using and learning more about them!
