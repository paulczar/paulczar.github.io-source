---
title: "Creating Self Signed Certificates on Kubernetes"
date: "2020-02-22"
categories: [kubernetes,certificates,ssl,tls,cert-manager]
---

Welcome to 2020. Creating self signed TLS certificates is still hard. Five (5) years ago I created a project on github called [omgwtfssl](https://github.com/paulczar/omgwtfssl) which is a fairly simple bash script wrapping a bunch of `openssl` commands to create certificates.

I've been using it ever since and kind of forgot about the pain of creating certificates.

*Skip the words and jump to the examples [Creating self signed certificates with cert-manager]  (#creating-self-signed-certificates-with-cert-manager), [Creating multiple certificates from the same self signed CA with cert-manager](#creating-multiple-certificates-from-the-same-self-signed-ca-with-cert-manager).*

With the advent of [letsencrypt](https://letsencrypt.org/) and later the Kubernetes [cert-manager](https://cert-manager.io/) controller we can make real signed certificates with a quick flourish of some **YAML**.

I've been happily chugging along with this combination of `cert-manager` [cert-manager](https://cert-manager.io/) for real certificates, and [omgwtfssl](https://github.com/paulczar/omgwtfssl) for self signed (despite the fact that the name is [inappropriate and unprofessional.](https://gitlab.com/gitlab-org/charts/gitlab/issues/584).

> "We should try to find a replacement for omgwtfssl, which is currently used to generate self-signed certificates. The name is inappropriate and unprofessional."  - [gitlab](https://gitlab.com/gitlab-org/charts/gitlab/issues/584)

As amusing as `docker run paulczar/omgwtfssl` is to type (I giggle every time), its a bit weird to tell people to create certificates locally then add them to their Kubernetes manifests or Helm charts. So I finally decided to sit down and figure out how to create them sensibly with [cert-manager](https://cert-manager.io/).

## Create a Kubernetes in Docker Cluster

You'll need a Kubernetes cluster, we're not doing anything too resource intensive so a [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) cluster should be fine.

Create [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) cluster:

```bash
kind create cluster
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```

Test the cluster:

```bash
kubectl cluster-info
```

## Creating self signed certificates with cert-manager

Install [cert-manager](https://cert-manager.io/):

```bash
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
```

> If you receive a validation error relating to the x-kubernetes-preserve-unknown-fields add `--validate` to the above command and run again.

Create a namespace to work in:

```bash
kubectl create namespace sandbox
```

Create an Issuer:

> Note: you can create a ClusterIssuer instead if you want to be able to request certificates from any namespace.

```bash
kubectl apply -n sandbox -f <(echo "
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
")
```

Create a self signed certificate:

> This creates a wildcard certificate that could be used for
  any services in the sandbox namespace.

```bash
kubectl apply -n sandbox -f <(echo '
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: first-tls
spec:
  secretName: first-tls
  dnsNames:
  - "*.sandbox.svc.cluster.local"
  - "*.sandbox"
  issuerRef:
    name: selfsigned-issuer
')
```

**Validate the secret is created**

Check the certificate resource:

```bash
$ kubectl -n sandbox get certificate
  NAME        READY   SECRET      AGE
  first-tls   True    first-tls   9s
```

Check the subsequent secret:

```bash
$ kubectl -n sandbox get secret first-tls
NAME        TYPE                DATA   AGE
first-tls   kubernetes.io/tls   3      73s
```

> This secret contains three keys `ca.crt`, `tls.crt`, `tls.key`. You can run `kubectl -n sandbox get secret first-tls -o yaml` to see the whole thing.

Test that the certificate is valid:

```bash
openssl x509 -in <(kubectl -n sandbox get secret \
  first-tls -o jsonpath='{.data.tls\.crt}' | base64 -d) \
  -text -noout
```

> If you scan through the output you should find `X509v3 Subject Alternative Name: DNS:*.first.svc.cluster.local, DNS:*.first`.

Congratulations. You've just created your first self signed certificate with Kubernetes. While it involves more typing than `docker run paulczar/omgwtfssl` it is much more useful for Kubernetes enthusiasts to have the cluster generate them for you.

However, what if you want to use TLS certificates signed by the same CA for performing client/server authentication? Never fear we can do that too.

## Creating multiple certificates from the same self signed CA with cert-manager

Install [cert-manager](https://cert-manager.io/):

**Skip this step if you already installed cert-manager from the first example.**

```bash
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
```

> If you receive a validation error relating to the x-kubernetes-preserve-unknown-fields add `--validate` to the above command and run again.

Create a namespace to work in:

```bash
kubectl create namespace sandbox2
```

Create an Issuer:

> Note: you can create a ClusterIssuer instead if you want to be able to request certificates from any namespace.

```bash
kubectl apply -n sandbox2 -f <(echo "
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
")
```

Create a CA Certificate:

> note `isCA` is set to true in the body of the `spec`.

```bash
kubectl apply -n sandbox2 -f <(echo '
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: sandbox2-ca
spec:
  secretName: sandbox2-ca-tls
  commonName: sandbox2.svc.cluster.local
  usages:
    - server auth
    - client auth
  isCA: true
  issuerRef:
    name: selfsigned-issuer
')
```

Check the certificate and secret were created:

```bash
$ kubectl -n sandbox2 get certificate sandbox2-ca
NAME          READY   SECRET            AGE
sandbox2-ca   True    sandbox2-ca-tls   15s

$ kubectl -n sandbox2 get secret sandbox2-ca-tls
NAME              TYPE                DATA   AGE
sandbox2-ca-tls   kubernetes.io/tls   3      22s
```

Create a second Issuer using the secret name from the `sandbox2-ca` secret:

> In order to sign multiple certificates from the same CA we need to create an Issuer resource from secret created by the CA.

```bash
kubectl apply -n sandbox2 -f <(echo '
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: sandbox2-ca-issuer
spec:
  ca:
    secretName: sandbox2-ca-tls')
```

Create a TLS Certificate from the new CA Issuer:

> We can add `usages` to the certificate `spec` to ensure that the certificates can be used for client/server authentication.

```bash
kubectl apply -n sandbox2 -f <(echo '
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: sandbox2-server
spec:
  secretName: sandbox2-server-tls
  isCA: false
  usages:
    - server auth
    - client auth
  dnsNames:
  - "server.sandbox2.svc.cluster.local"
  - "server"
  issuerRef:
    name: sandbox2-ca-issuer
')
```

Create a second TLS Certificate from the new CA Issuer:

```bash
kubectl apply -n sandbox2 -f <(echo '
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: sandbox2-client
spec:
  secretName: sandbox2-client-tls
  isCA: false
  usages:
    - server auth
    - client auth
  dnsNames:
  - "client.sandbox2.svc.cluster.local"
  - "client"
  issuerRef:
    name: sandbox2-ca-issuer
')
```

Check that all three certificates are created:

```bash
$ kubectl -n sandbox2 get certificate
NAME              READY   SECRET                AGE
sandbox2-ca       True    sandbox2-ca-tls       7m34s
sandbox2-client   True    sandbox2-client-tls   7s
sandbox2-server   True    sandbox2-server-tls   16s

$ kubectl -n sandbox2 get secret
NAME                  TYPE                                  DATA   AGE
sandbox2-ca-tls       kubernetes.io/tls                     3      8m14s
sandbox2-client-tls   kubernetes.io/tls                     3      48s
sandbox2-server-tls   kubernetes.io/tls                     3      57s
```

Validate the certificates against the CA:

```bash
$ openssl verify -CAfile \
<(kubectl -n sandbox2 get secret sandbox2-ca-tls \
  -o jsonpath='{.data.ca\.crt}' | base64 -d) \
<(kubectl -n sandbox2 get secret sandbox2-server-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d)
/proc/self/fd/18: OK

$ openssl verify -CAfile \
<(kubectl -n sandbox2 get secret sandbox2-ca-tls \
  -o jsonpath='{.data.ca\.crt}' | base64 -d) \
<(kubectl -n sandbox2 get secret sandbox2-client-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d)
/proc/self/fd/18: OK
```

**Validate the Client / Server authentication**

Run an `openssl` server as a background process:

```bash
touch test.txt

openssl s_server \
  -cert <(kubectl -n sandbox2 get secret sandbox2-server-tls -o jsonpath='{.data.tls\.crt}' | base64 -d) \
  -key <(kubectl -n sandbox2 get secret sandbox2-server-tls -o jsonpath='{.data.tls\.key}' | base64 -d) \
  -CAfile <(kubectl -n sandbox2 get secret sandbox2-ca-tls -o jsonpath='{.data.ca\.crt}' | base64 -d) \
  -WWW -port 12345  \
  -verify_return_error -Verify 1 &
```

Run an `openssl` client test:

*look for `HTTP/1.0 200 ok` in the client output.*

```bash
echo -e 'GET /test.txt HTTP/1.1\r\n\r\n' | \
  openssl s_client \
  -cert <(kubectl -n sandbox2 get secret sandbox2-client-tls -o jsonpath='{.data.tls\.crt}' | base64 -d) \
  -key <(kubectl -n sandbox2 get secret sandbox2-client-tls -o jsonpath='{.data.tls\.key}' | base64 -d) \
  -CAfile <(kubectl -n sandbox2 get secret sandbox2-client-tls -o jsonpath='{.data.ca\.crt}' | base64 -d) \
  -connect localhost:12345 -quiet
```

stop the background process:

```bash
kill %1
```

Congratulations, you've now created a pair of certificates signed by the same CA that can be used for client/server authentication.

## Conclusion

Creating self signed certificates is now officially easy. You can use [omgwtfssl](https://github.com/paulczar/omgwtfssl) locally, or [cert-manager](https://cert-manager.io/) in your Kubernetes cluster. Either way you get cheap and easy self signed certificates for testing. Obviously you should use real certificates in production, in which case you would still be able to use [cert-manager](https://cert-manager.io/).