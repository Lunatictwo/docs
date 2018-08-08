---
title: Deploy TiDB to Kubernetes on your laptop
summary: Deploy TiDB to Kubernetes on your laptop
category: operations
---


# Deploy TiDB, a distributed MySQL compatible database, to Kubernetes on your laptop

[TiDB](www.pingcap.com) is a scale-out distributed MySQL-compatible database.
Scale-out means that when you have high usage of CPU, RAM, or disk, you just add another node to your cluster and the cluster's capacity will increase accordingly.
This is much easier to administer at scale, but we need deployment to be simple from day one. We have an [Ansible-based deployment](https://www.pingcap.com/docs/op-guide/ansible-deployment/) that works in almost any environment. However, if a TiDB cluster is deployed using Kubernetes, we can provide an even better experience.

For this tutorial we will deploy TiDB using Kubernetes on our laptop, but this can be done on any Kubernetes cluster.


## TIDB Architecture

Let's review what we will be deploying.

* TiKV
* Placement Driver (PD)
* TiDB stateless SQL layer

The scale-out property of TiDB is provided by [TiKV](https://github.com/pingcap/tikv), a distributed transactional key-value store.
The TiKV cluster itself requires deploying a PD cluster. PD stands for placement driver, which controls where data is stored in TiKV.
MySQL compatibility is provided by a separately deployed TiDB SQL component that's stateless and works on top of TiKV.


## Kubernetes architecture

TiKV and PD maintain database state on disk, and are thus mapped to a [StatefulSet]() with a PersistentVolume Claim.
TiDB SQL is also mapped to a StatefulSet, but it does not make any PersistentVolume Claims.

These are wrapped together in a helm chart called tidb-cluster.
Additionally, we provide [tidb-operator](), a Kubernetes operator. This program monitors the status of TiDB in your Kubernetes cluster and provides a gateway to administrative duties.


### Installing a Kubernetes cluster on your laptop

This installation has been tested to work on Mac and Linux, but please note that running Kubernetes on a laptop is a work in progress.

First we need to run a Kubernetes (k8s) cluster. minikube is the popular option for that. However, minikube only creates one Kubernetes node. To run TiDB, we need multiple Kubernetes nodes. There are a few options for this, but for this tutorial we will use DinD (Docker in Docker).

DinD (Docker in Docker) allows for running the docker daemon inside a top-level docker container. This means the top-level container can simulate as a Kubernetes node and have containers launched inside it. The kubeadm-
dind-cluster project starts multiple docker containers as a k8s node on a standalone machine through DinD, and then start a k8s cluster by using docker to start k8s components on these nodes.

First [install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

Then, make sure Docker is running on your laptop. Install Docker [here](https://docs.docker.com/install/) if you haven't done so.

Next, we will bring up the DinD K8s. First we need to make sure that `sha1sum` installed:

    which sha1sum || brew install md5sha1sum

Then we can bring the cluster up

    git clone https://github.com/pingcap/kubeadm-dind-cluster.git
    cd kubeadm-dind-cluster
    DIND_USE_NFS_STORAGE=false NUM_NODES=4 ./tools/multi_k8s_dind_cluster_manager.sh rebuild e2e-v1.10

Launching this will take a while, a good time to stretch and drink some water.
When you are back, and the install finishes, ensure that the DinD cluster works:

    kubectl get node,componentstatus 
    kubectl get pod -n kube-system

If you would like, you can now [view the dashboard](http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/) once you start the proxy with:

    kubectl proxy

Local Persistent volumes don't work right in DinD so we need to provision them manually ourselves

    cd ..
    git clone https://github.com/pingcap/tidb-operator
    # temporary directory until repo switch happens
    cd tidb-operator/new-operator
    git fetch origin
    git checkout gregwebs/setup-scripts
    ./manifests/local-dind/setup.sh

Note that it may take some time to pull the docker image for local volume provisioning.

## Running TiDB

This process is the same regardless of how you create a Kubernetes cluster. First we launch the operator:

You need to have helm installed and tiller running as tested by `helm ls`. The DinD installer above will install helm. But you can also install it with:

    curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > helm.sh
    bash helm.sh

Make sure helm is initialized.

    helm init --upgrade

Use helm to launch tidb-operator. Note that you may need to wait a few additional seconds for `helm init` to finish its deployment.

    cd ..
    git clone https://github.com/pingcap/private-tidb-operator
    cd tidb-operator/new-operator
    helm install charts/tidb-operator --name tidb-operator --namespace=tidb-operator

Now we need to wait for the operator to come up. This can be done with:

    kubectl get pods --namespace tidb-operator -l app=tidb-operator

Now we can launch TiDB itself:

    kubectl apply -f manifests/crd.yaml
    helm install charts/tidb-cluster --name tidb-cluster --namespace=tidb

You now have a distributed database running on Kubernetes! It should take less than a minute for all the services to come up, but you can watch this process with:

    watch kubectl get pods --namespace tidb

TODO: screenshot of proper output.

We can connect to it with MySQL

    kubectl port-forward svc/demo-cluster-tidb 4000:4000 -n tidb
    mysql -u root -h 127.0.0.1 -P 4000

If you don't have MySQL installed, you can run it from a container:

    docker run --rm -it --net host jbergknoff/mysql-client mysql -u root -h 127.0.0.1 -P 4000

Similarly, you can open the [Grafana dashboard](http://localhost:3000/dashboard/db/tidb-cluster-pd) for operational metrics after forwarding the grafana port

    kubectl port-forward svc/demo-cluster-grafana 3000:3000 -n tidb


Lets add some data

	wget http://download.pingcap.org/tispark-sample-data.tar.gz
	tar zxvf tispark-sample-data.tar.gz
	cd tispark-sample-data
	mysql --local-infile=1 -u root -h 127.0.0.1 -P 4000 < dss.ddl

Now you can connect to MySQL again and look at your data:

	USE TPCH_001;
	SHOW TABLES;
	select * from nation;


## Cleaning up

The installer script for DinD comes with its own clean command to wipe out the laptop Kubernetes cluster.

    NUM_NODES=4 ./dind-cluster-v1.11.sh clean.

If you want to keep your cluster running, you can un-install the helm charts.
To avoid data loss, persistent volumes and their claims are maintained and also need to be deleted

    helm delete --purge tidb-operator tidb-cluster
    kubectl get pv -o name | xargs -i kubectl patch {} -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
    kubectl delete pvc -n tidb --all
