---
title: "What's cheaper - DB on K8S or AWS RDS?"
date: 2018-03-24T11:07:10+06:00
author: Bill Shetti
draft: true
readtime: "5 Min"
description: ""
featuredImage: "/images/articles/k8s_cost_ch-bs/featuredimage.png"
bannerImage: "/images/articles/k8s_cost_ch-bs/historical.png"
articleDisplayCategory: "Multi Cloud Operations"
featured: false
categories: [multicloudops, bill]
tags: [Kubernetes,Database,DevOps,CloudHealth,AWS]
aliases:
    - /blog/db_cost_aws-bs/
---

As more and more applications convert to cloud native with Kubernetes as a normalizing layer the ability to have portability between public clouds also becomes easier. Theoretically, the app can now move easily between AWS/Azure/GCP. However, a couple of issues are always in the back of developer’s minds: 
 
* Do I connect to cloud services like ML/AI, DevOps tool chains, DBs, Storage, etc? This might lock me in.
* Should I roll my own database or use a cloud services like CosmosDB, Azure Postgres DB, etc.?
 
I covered the complexity of the second question in a previous article. [Application state what is easier?](https://www.cloudjourney.io/articles/devops/application-state-whats-easier-bs/)

Today I will cover what the cost differential between rolling your own on K8S vs using a service like RDS from AWS. 

In order to show case the differences, I will use the [Acme Fitness Shop App](https://github.com/vmwarecloudadvocacy/acme_fitness_demo) deployed in two Kubernetes configurations:

* Deploy the e-commerce application with Portworx on the Kubernetes cluster. Portworx will support volumes in AWS EC2 which will give persistence storage, and other capabilities like DR, [backup](https://portworx.com/kubernetes-backup/), security, etc.
* Deploy the e-commerce application with RDS. This is generally the logical choice since its easy to bring up and manage on AWS.

I'll also use **Postgres** on both with similar configurations to properly compare them. 

In [Acme Fitness Shop App](https://github.com/vmwarecloudadvocacy/acme_fitness_demo) there are multiple services connecting to different databases. For this article, I only varied the Order Service configuration, which connects to Postgres. Hence, one version uses Portwrox + postgres operator, and the other AWS RDS+Postgres.

So, how do we compare the costs? We used Cloudhealth for comparing the two configurations.

## Some component basics and configuration:

A quick review of the main components used in this blog:

1.  **Cloudhealth:** Used to calculate the costs of each configguration. Its a SaaS service from VMware that provides an ability to easily manage cost, ensure security compliance, improve governance and automate actions across multi-cloud environments. Known for offering the highest levels of data integrity throughout an organization’s entire cloud journey, CloudHealth is the platform of choice for leading enterprises and service providers.


2.  **Portworx** - The Portworx Enterprise Storage Platform is your end-to-end storage and data management solution for all your Kubernetes projects. Portworx provides a slew of [features](https://portworx.com/products/features/) that include high availability, disaster recovery, backup and restore, security, automated capacity management, multi-cluster management and migration of application and data that those applications are using. The benefit of Portworx on AWS is the abstraction of underlying EBS such that every volume an application uses is optimized for cost and abstracted to provide the best experience without sacrificing performance or flexibility. In fact, in most cases, Kubernetes used will have an [improved operations](https://portworx.com/ebs-stuck-attaching-state-docker-containers/) and [better economies of scale](https://portworx.com/architects-corner-aurea-beyond-limits-amazon-ebs-run-200-kubernetes-stateful-pods-per-host/). 

![](px-arch.png)

In this setup, each Portworx node in the EKS cluster was configured with a single EBS volumes using [automated volume templates](https://docs.portworx.com/cloud-references/auto-disk-provisioning/aws/#1-using-a-template-specification). Portworx then pools these together to allow applications on the K8S cluster to create virtual thin-provisioned, highly available Persistent Volumes (PVs) which support the Postgres deployment.


3. **AWS EKS** - Elastic Kubernetes Service - Two exactly similar clusters were configured with the following config using EKS CTL

```
eksctl create cluster --name acme-portworx --region us-east-1 --node-type t3.large --nodes 3
```

4. **postgres-operator** - There are several Postgres Operators available. I used the [zalando/postgres-operator](https://github.com/zalando/postgres-operator). This enabled deployment of Postgres on to the Portworx volumes. Postgres was configured to have a single writer and two read replicas (Just like RDS) across 3 availability zones (one on each EKS node in AZ a,b,c)

5. **AWS RDS** - was configured for Postgres with a MultiAz configuration with 200G capacity. (same as Portworx)

6. **[Acme Fitness Shop App](https://github.com/vmwarecloudadvocacy/acme_fitness_demo)** - There are two options for the acme fitness shop. Per the image below, only order's postgres db is varied in configuration.

![Postgres options](/images/articles/db_cost_aws-bs/acmeshop-postgres-options.png)

As part of the configuration I also did not configure backups nor run any loads on the configuration in order to keep the comparison simple. 

## What's the cost differential?

Before we walk through the detailed configuration and setup, its useful to see the output
Cloudhealth provides in analyzing either configuration.

Both configurations were run for 5 days. In order to properly collect data on either configuration, two filters (perspectives in cloud health) were set up to properly collect data on the relevant components in each configuration. (I'll review the configuration details in a later section.)

Since the logical configuration most people would use is RDS, lets start with the RDS configuration.

### RDS

![RDS-Chart](/images/articles/db_cost_aws-bs/RDS-chart.png)

![RDS-Table](/images/articles/db_cost_aws-bs/RDS-table.png)

Because EKS clusters are a combination of multiple components, the EKS cluster in this configuration is made up of EC2 isntances and  EBS storage (volumes).  The cost of running an EKS cluster is $6.18/day.

The application requires a load balancer, and the data transfer into and out of the application is only $0.61/day.

RDS, on the other hand uses EC2, EBS, data transfer, and replicas. These are all noted with RDS label per line in the table. RDS runs $15.01/day.

The total cost of the configuration runs $21.81/day.

The cost of running  RDS  is roughly ~2.5x the cost of running an EKS cluster.

### Portworx

![Portworx-Chart](/images/articles/db_cost_aws-bs/portworx-chart.png)

![Portworx-Table](/images/articles/db_cost_aws-bs/portworx-table.png)

Because EKS clusters are a combination of multiple components, the EKS cluster in this configuration is made up of EC2 isntances and  EBS storage (volumes).  

Portworx adds volumes to help support the installation of a database, hence the cost of EBS storage is $3.58/day. This consists of 9 volumes, 6 of which are configured by Portworx, and 3 are root volumes for EKS cluster nodes.

Let's review the breakdown of the volume costs.

![Portworx-Volumes](/images/articles/db_cost_aws-bs/portworx-volumes.png)

If we just take the baseline EC2 (EKS Cluster) volumes ($.06/day), then the total cost of the EKS cluster (EC2 + EBS) is $6.18/day - SAME as the RDS cluster. Portworx adds $3.40/day of volume costs. Oddly this additional cost is still minimal addition to the total/day cost for the Portworx configuration. At $10.81/day the Portworx configuration is 50% cheaper than the RDS configuration. 

### Portworx + Postgres or RDS?

Based on the cost comparison above, its obvious that Portworx+Postgres is the better option at 50% cheaper to operate than RDS. However, in any operations organization there are many parameters that help make the decision to use a managed service vs self built. Cost is an important aspect, but parameters like K8S expertise or even DB expertise are important considerations. If we assume K8S expertise and DB expertise is in house then Portworx+Postgres is the best option to help reduce the operational cost of the solution. 

## Detailed configurations

If you want to replicate this analysis, lets break down the details across the following components:

1. K8S Cluster configuration (for either cluster)
2. Application deployment on the clusters
3. Portworx detailed configuration + Postgres Operator
4. RDS configuration
5. Cloudhealth configuration

### AWS EKS Configuration

As noted earlier both EKS clusters are configured using EKS CTL with the following configuration

```
eksctl create cluster --name acme-portworx --region us-east-1 --node-type t3.large --nodes 3
```

This configuration will bring up the following configuration in EC2 for each EKS Cluster

* 3 EC2 t3.large nodes
* 3 EBS Volumes (non-encrypted) with 20GBs each attached to a corresponding EC2 instance. Each volume is gp2 with 100 IOPS


### App Deployment

The app is deployed with one of the two configurations noted above (see image below)

![Postgres options](/images/articles/db_cost_aws-bs/acmeshop-postgres-options.png)

In order to properly configure the app to either option, its as simple as pointing the postgres host container variable to the proper end point.

Here is a snippet of the K8S yaml file configuring the postgres end point for the order service.

```
    spec:
      volumes:
      - name: acmefit-order-data
        emptyDir: {}
      containers:
      - image: gcr.io/vmwarecloudadvocacy/acmeshop-order:latest
        # orderpostgres:latest
        name: order
        env:
        - name: ORDER_DB_HOST
          value: pwx-pg-cluster
        - name: ORDER_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-postgres-pass
              key: password
        - name: ORDER_DB_PORT
          value: '5432'
        - name: AUTH_MODE
          value: '1'
        - name: ORDER_DB_USERNAME
          value: postgres
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: order-postgres-pass
              key: password
        - name: ORDER_AUTH_DB
          value: postgres
```

As noted above the following ENV Variables are different in each configuration:

| ENV  | Portworx  | RDS  |
|---|---|---|
| ORDER_DB_HOST  | pwx-pg-cluster  | acme-postgres.XXXXX.us-east-1.rds.amazonaws.com  |
| ORDER_DB_PASSWORD  | YYYY  | YYYY  |
| ORDER_DB_PORT  | 5432  | 5432  |

These three ENV Variables are the only ENV variables to that vary between each of the DB end points between the Portworx or the RDS configuration.

The application also brings up an ELB in order to allow access to the front end.

```
ubuntu@ip-172-31-35-91:~/portworx/postgres-operator/manifests$ kubectl get services
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)          AGE
cart                    ClusterIP      10.100.70.164    <none>                                                                   5000/TCP         10d
cart-redis              ClusterIP      10.100.207.165   <none>                                                                   6379/TCP         10d
catalog                 ClusterIP      10.100.160.44    <none>                                                                   8082/TCP         10d
catalog-mongo           ClusterIP      10.100.112.56    <none>                                                                   27017/TCP        10d
frontend                LoadBalancer   10.100.157.239   aaa64accc680d11eab98602126feabd1-802863958.us-east-1.elb.amazonaws.com   80:31904/TCP     10d
order                   ClusterIP      10.100.198.156   <none>                                                                   6000/TCP         10d
payment                 ClusterIP      10.100.75.63     <none>                                                                   9000/TCP         10d
users                   ClusterIP      10.100.180.95    <none>                                                                   8083/TCP         10d
users-mongo             ClusterIP      10.100.14.28     <none>                                                                   27017/TCP        10d
users-redis             ClusterIP      10.100.3.12      <none>                                                                   6379/TCP         10d
```

### Portworx + Postgres

#### Portworx setup

Portworx is set up with a spec that is generated through a UI located at [PX-Central](https://central.portworx.com/dashboard). 

The configuration is set up with the following parameters:
1. PWX Internally Hosted etcd - this allows Portworx to run a highly available version of etcd for the cluster increasing the availability of the cluster
2. AWS is selected as the cloud
3. GP2 based EBS volume types selected with 200GB capacity

This will bring up a single Portworx cluster EBS in the following ways:
1. 3 redundant volumes (1 per EKS node) which Portworx pools together into a single virtual storage pool. Postgres uses this pool to create [PVs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) for persistence.
2. 3 redundant volumes in the cluster to support a 3 node etcd cluster @ 150GB per volume for Portworx metadata.

The following is should pop up in the kube-system K8S namespace:

```
kube-system   portworx-5dfjm                            1/1     Running   0          11d
kube-system   portworx-api-7nglm                        1/1     Running   0          59d
kube-system   portworx-api-k7fhw                        1/1     Running   0          59d
kube-system   portworx-api-qvmwm                        1/1     Running   0          11d
kube-system   portworx-fsk84                            1/1     Running   0          59d
kube-system   portworx-h6jkn                            1/1     Running   0          59d
kube-system   portworx-pvc-controller-cc68c78f9-bh9vg   1/1     Running   0          59d
kube-system   portworx-pvc-controller-cc68c78f9-vmdpf   1/1     Running   2          59d
kube-system   portworx-pvc-controller-cc68c78f9-zr5v2   1/1     Running   0          11d
kube-system   px-lighthouse-6d48b4df85-wr72s            3/3     Running   0          59d
```

#### Postgres setup

We used the following [postgres operator](https://github.com/zalando/postgres-operator) from a company called Zalando. To understand the components, take a look at [the docs](https://postgres-operator.readthedocs.io/en/latest/#overview-of-involved-entities).

The setup is simple from the installation instructions, but what it actually brings up is the following:

- Postgres [Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) (Operator and API to setup Postgres)
- Postgres Operator UI (Optional: a UI for Postgres Clusters in K8s)
- 1 Master (Writer) pwx-pg-cluster-0
- 2 Replicas (Reader) pwx-pg-cluster-1, pwx-pg-cluster-2

```
PODS:

default       postgres-operator-6d9f4d577f-fsn22        1/1     Running   0          11d
default       postgres-operator-ui-774b77594-rg7zj      1/1     Running   0          37d
default       pwx-pg-cluster-0                          1/1     Running   0          11d
default       pwx-pg-cluster-1                          1/1     Running   0          37d
default       pwx-pg-cluster-2                          1/1     Running   0          37d

Service:
pwx-pg-cluster          LoadBalancer   10.100.191.46
```

You'll notice that the `pwx-pg-cluster` also brings up an ELB. This is done to allow access to this cluster as needed from any location. 
The `pwx-pg-cluster-x` is the postgres implementation using the volumes brought up with Portworx.

Connecting to the Postgres DB is as simple as pointing the app's `ORDER_DB_HOST` to `pwx-pg-cluster`. See section above

Portworx can retain internal snapshots using a snapshot [schedule policy](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/scheduled/#creating-a-schedule-policy). This does not add to the cost as these snapshots are using the same storage pool.

### RDS configuration

In keeping normalized with the portworx+postgres configuration, AWS RDS was configured with the following parameters:

* [Multi-AZ](https://aws.amazon.com/rds/features/multi-az/) was turned on - similar to the postgres cluster in the portworx+postgres config. This RDS configuration has 1 master and 2 [read replicas](https://aws.amazon.com/rds/features/read-replicas/) as well.
* EC2 instance type used was db.t3.large, similar to what is used in the K8S cluster
* Storage is 200GB as with the portworx-postgres configuration
* Snapshots are configured to be deleted after 1 day
* No backups or backup storage

Configuring RDS in the app is as simple as pointing the `ORDER_DB_HOST` to `acme-postgres.XXXXX.us-east-1.rds.amazonaws.com`


### Cloudhealth configuration

Configuring Cloudhealth is fairly simple. It requires setting up something called **Perspectives** for each configuration.
**Perspectives** are essentially a filter pinpointing specific resources for analysis.


Portworx Perspective consists of the following assets/resources in AWS

1. 3 EC2 t3.large nodes for the EKS cluster
2. 9 EBS volumes 
* 3 EBS Volumes (non-encrypted) with 20GBs each attached to a corresponding EC2 instance. Each volume is gp2 with 100 IOPS plus
* 3 redundant volumes in a cluster to support etcd for the cluster @ 150GB per volume
* 3 EBS volumes connected to the 3 EKS EC2 connected volumes at 200GB each for Portworx Storage Pool
3. 2 ELBs One for the application and one for postgres 

RDS Perspective consists of the following assets/resources in AWS

* 6 EC2 instances - 3 t3.large instances for the EKS cluster and 3 for Multi-Zone RDS instance
* 3 EBS volumes connected to the 3 EKS EC2 connected volumes at 20GB each
* 3 EBS volumes too support Postgres RDS nodes @ 200GB each
* 1 ELB supporting the application front end


## Conclusion

Hopefully the analysis above has provided an overview of how you can potentially save on costs through self configuration and management of a postgres db in kubernetes. 
Portworx provides a simple mechanism to essentially build the infrastructure needed to load up any database. Although we only used the most 
basic of configurations, there are many more features that can be turned on with Portworx. If you have the DB expertise and are adept in managing K8S clusters and configurations,
Portworx is the way to go.

Saving 50% in cost over AWS RDS is a bonus and a strong consideration for this option. 

Checkout more from [Portworx](https://portworx.com).
