---
title: "Deploying and accessing a GCP SQL database via Kubernetes"
description: ""
categories: ["kubernetes", "k8s", "gcp", "google cloud", "google config connector"]
date: "2020-06-08"
draft: false
---

*Sorry for the long preamble, feel free to just skip straight to the [technical details]({{< relref "#deploying-a-cloudsql-instance-via-gcc" >}}).*

Two years ago, faced with the challenge of wanting to reduce the tooling required to deploy infrastructure to Google Cloud I started working on a [Google Cloud Operator](https://github.com/paulczar/gcp-cloud-compute-operator). The goal of this project was to be able to stand up networks, images, and Virtual Machines for installing Pivotal's [PKS](https://docs.pivotal.io/pks/1-4/index.html) (now Tanzu Kubernetes Grid Integrated).

This worked out pretty well for me and I was able to create/destroy environments at will seeded from a single node GKE cluster running the operator.

Thankfully later some folks at Google started working on an Operator of their own which they named [Google Config Connector](https://cloud.google.com/config-connector/docs/overview) which sought to solve roughly the same problem. Although their original intent was more around backing services for Kubernetes workloads such as databases.

As they brought in additional features I was able to start offloading some of the work my own [operator was doing to GCC](https://www.youtube.com/watch?v=XL-icNS-IEg&t=38s), however only recently did they support the full set of services that I require.

Recently I've been doing some work around running applications on Kubernetes and been thinking about how I want to handle databases in Dev vs Prod. Using a tool like [Helm](https://helm.sh) or [ytt](https://get-ytt.io/) I could in theory have a flag that determines whether it should create a Postgres deployment in Kubernetes, or a [CloudSQL](https://cloud.google.com/sql) instance.

I hadn't actually used the Google Config Connector operator to stand up a Database to be accessed from inside a GKE cluster, so I figured that would be the natural place to start.

Unfortunately I found [gaps](https://github.com/GoogleCloudPlatform/k8s-config-connector/issues/201) in both the [GCC examples](https://github.com/GoogleCloudPlatform/k8s-config-connector/tree/master/samples/resources/sqlinstance/private-ip-instance) and the [CloudSQL documentation](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine).

I could get the database running easily enough but I just could not get applications running on GKE to access it over the Private IP. Finally I found this statement `For connecting using private IP, the GKE cluster must be VPC-native and in the same VPC network as the Cloud SQL instance.`.

Doing some digging it became apparent that GKE clusters are not [VPC-native](https://cloud.google.com/kubernetes-engine/docs/how-to/alias-ips) by default. I also found that SQL Instances actually never live in your VPC and you need to allocate an IP Range to peering in that network and set up a [Service Network Connection](https://cloud.google.com/vpc/docs/configure-private-services-access#creating-connection).

With that figured out I was able to successfully create a new GKE cluster, install GCC, and deploy and use a Cloud SQL instance over its private IP.

## Deploying a CloudSQL instance via GCC

If you want to refer to a `ytt` templated version of the manifests created below you can find a fully working example (that also includes Google SQL Proxy) [here](https://github.com/paulczar/gcc-cloudsql).

### Create GKE Cluster

Before we do anything we need to ensure a few of the Google Cloud APIs are enabled.

```bash
gcloud services enable \
  servicenetworking.googleapis.com \
  servicemanagement.googleapis.com \
  iamcredentials.googleapis.com
```

Some of our commands require your Google Cloud Project ID, you can find it and save it as a variable for later use like so:

```bash
PROJECT_ID=$(gcloud config get-value project)
```

Create a Small GKE Cluster using `--workload-pool` to enable workload identity and `--enable-ip-alias` to create a VPC-native cluster:

```bash
gcloud container clusters create gcc-cloudsql \
  --num-nodes=1 --zone us-central1-c \
  --cluster-version 1.16 --machine-type n1-standard-2 \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --enable-ip-alias
```

Check the cluster is accessible:

```bash
kubectl cluster-info
```

In order for Applications in GKE to access the Private IP of CloudSQL instances you need to create a VPC Peering Range and a Service Networking Connection.

*This only has to be done once per Network in your account.*

Create a VPC Peering range:

```bash
gcloud compute addresses create cloudsql-peer \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --description="peering range for CloudSQL" \
    --network=default \
    --project=$PROJECT_ID
```

Peer that range with our default network:

```bash
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=cloudsql-peer \
    --network=default \
    --project=$PROJECT_ID
```

*If this command fails with Cannot modify allocated ranges in CreateConnection then rerun the command but replace `connect` with `update --force`.*

### Deploy Google Config Connector

Create a service account for GCC:

```bash
gcloud iam service-accounts create cnrm-system
```

Bind the roles/owner to the service account:

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/owner"
```

Bind roles/iam.workloadIdentityUser to the cnrm-controller-manager Kubernetes Service Account in the cnrm-system Namespace:


```bash
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com
```

Download GCC:

```bash
gsutil cp gs://cnrm/latest/release-bundle.tar.gz release-bundle.tar.gz
```

Extract GCC:

```bash
tar zxvf release-bundle.tar.gz
```

Fix the manifest with your project id:

```bash
sed -i.bak "s/\${PROJECT_ID?}/${PROJECT_ID}/" \
  install-bundle-workload-identity/0-cnrm-system.yaml
```

Apply the manifest to your cluster:

```bash
kubectl apply -f install-bundle-workload-identity/
```

Wait until GCC is fully online:

```bash
kubectl wait -n cnrm-system --for=condition=Ready pod --all
```

### Deploy CloudSQL Instance

Create a Namespace:

```bash
kubectl create namespace $PROJECT_ID
```

Create the Cloud SQL Instance:

```bash
kubectl -n $PROJECT_ID apply -f <(cat << EOF
apiVersion: sql.cnrm.cloud.google.com/v1beta1
kind: SQLInstance
metadata:
  name: example
spec:
  region: us-central1
  databaseVersion: POSTGRES_9_6
  settings:
    tier: db-custom-1-3840
    ipConfiguration:
      ipv4Enabled: false
      requireSsl: false
      privateNetworkRef:
        external: default
EOF
)
```

Create a Cloud SQL User:


```bash
kubectl -n $PROJECT_ID apply -f <(cat << EOF
apiVersion: sql.cnrm.cloud.google.com/v1beta1
kind: SQLUser
metadata:
  name: example
spec:
  instanceRef:
    name: example
  password:
    value: "bad-password"
EOF
)
```

After a few moments check that the SQL Instance is being created:

```bash
watch gcloud sql instances list
```

Once the `STATUS` changes from `PENDING_CREATE` to `RUNNABLE` hit CTRL-C to exit the `watch` command.

We should now be able to confirm that an Application running in the GKE cluster can access the database. To validate this we can use the `postgres:13-alpine` image.

Run the postgres image as a Pod with an interactive shell:

```bash
kubectl run -ti --restart=Never --image postgres:13-alpine --rm psql -- sh
```

Inside that shell run `psql`:

*Change the IP to match the IP from the `gcloud sql instances list` command. Enter the password `bad-password` when prompted*

```bash
$ psql -h ip.add.re.ss --username=example -d postgres
Password for user example: bad-password
psql (13beta1, server 9.6.16)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```

## Conclusion

That's it! Super easy and powerful way to get a managed database provisioned and usable without ever leaving Kubernetes.

If you want to make things even more Kubernetes friendly you can use Google's [SQL Proxy](https://github.com/paulczar/gcc-cloudsql/blob/master/ytt/cloudsql/proxy.yaml) inside your Kubernetes cluster which helps manage your SSL Connections to the databases as well as being able to use predictable names rather than hunting for IP addresses.
