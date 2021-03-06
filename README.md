# Pravega Operator

 [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![GoDoc](https://godoc.org/github.com/pravega/pravega-operator?status.svg)](https://godoc.org/github.com/pravega/pravega-operator) [![Build Status](https://travis-ci.org/pravega/pravega-operator.svg?branch=master)](https://travis-ci.org/pravega/pravega-operator) [![Go Report](https://goreportcard.com/badge/github.com/pravega/pravega-operator)](https://goreportcard.com/report/github.com/pravega/pravega-operator) [![Version](https://img.shields.io/github/release/pravega/pravega-operator.svg)](https://github.com/pravega/pravega-operator/releases)

### Project status: alpha

The project is currently alpha. While no breaking API changes are currently planned, we reserve the right to address bugs and change the API before the project is declared stable.

## Table of Contents

 * [Overview](#overview)
 * [Requirements](#requirements)
 * [Quickstart](#quickstart)    
 * [Install the Operator](#install-the-operator)
 * [Deploying in Test Mode](#deploying-in-test-mode)
 * [Upgrade the Operator](#upgrade-the-operator)
 * [Install a sample Pravega Cluster](#install-a-sample-pravega-cluster)
 * [Scale a Pravega Cluster](#scale-a-pravega-cluster)
 * [Upgrade a Pravega Cluster](#upgrade-a-pravega-cluster)
 * [Uninstall the Pravega Cluster](#uninstall-the-pravega-cluster)
 * [Uninstall the Operator](#uninstall-the-operator)
 * [Manual installation](#manual-installation)
 * [Configuration](#configuration)
 * [Development](#development)
 * [Releases](#releases)
 * [Troubleshooting](#troubleshooting)

## Overview

[Pravega](http://pravega.io) is an open source distributed storage service implementing Streams. It offers Stream as the main primitive for the foundation of reliable storage systems: *a high-performance, durable, elastic, and unlimited append-only byte stream with strict ordering and consistency*.

The Pravega Operator manages Pravega clusters deployed to Kubernetes and automates tasks related to operating a Pravega cluster.

- [x] Create and destroy a Pravega cluster
- [x] Resize cluster
- [x] Rolling upgrades

## Requirements

- Kubernetes 1.15+
- Helm 3.2.1+
- An existing Apache Zookeeper 3.6.1 cluster. This can be easily deployed using our [Zookeeper operator](https://github.com/pravega/zookeeper-operator)
- An existing Apache Bookkeeper 4.9.2 cluster. This can be easily deployed using our [BookKeeper Operator](https://github.com/pravega/bookkeeper-operator)

## Quickstart

We recommend using our [helm charts](charts) for all installation and upgrades (but not for rollbacks at the moment since helm rollbacks are still experimental). The helm charts for pravega operator (version 0.4.5 onwards) and pravega cluster (version 0.5.0 onwards) are published in [https://charts.pravega.io](https://charts.pravega.io/). To add this repository to your Helm repos, use the following command
```
helm repo add pravega https://charts.pravega.io
```
There are manual deployment, upgrade and rollback options available as well.

### Install the Operator

> Note: If you are running on Google Kubernetes Engine (GKE), please [check this first](doc/development.md#installation-on-google-kubernetes-engine).

To understand how to deploy a Pravega Operator using helm, refer to [this](charts/pravega-operator#installing-the-chart).

#### Deploying in Test Mode
 The Operator can be run in `test mode` if we want to deploy pravega on minikube or on a cluster with very limited resources by enabling `testmode: true` in `values.yaml` file. Operator running in test mode skips minimum replica requirement checks on Pravega components. Test mode provides a bare minimum setup and is not recommended to be used in production environments.

### Upgrade the Operator

For upgrading the pravega operator check the document [operator-upgrade](doc/operator-upgrade.md)

### Install a sample Pravega cluster

#### Set up Tier 2 Storage

Pravega requires a long term storage provider known as longtermStorage.

Check out the available [options for long term storage](doc/longtermstorage.md) and how to configure it.

For demo purposes, you can quickly install a toy NFS server.

```
$ helm repo add stable https://charts.helm.sh/stable
$ helm repo update
$ helm install stable/nfs-server-provisioner --generate-name
```

And create a PVC for longtermStorage that utilizes it.

```
$ kubectl create -f ./example/pvc-tier2.yaml
```

#### Install a Pravega cluster

To understand how to deploy a pravega cluster using helm, refer to [this](charts/pravega#installing-the-chart).

Once the pravega cluster with release name `bar` has been created, use the following command to verify that the cluster instances and its components are being created.

```
$ kubectl get PravegaCluster
NAME          VERSION   DESIRED MEMBERS   READY MEMBERS   AGE
bar-pravega   0.4.0     7                 0               25s
```

After a couple of minutes, all cluster members should become ready.

```
$ kubectl get PravegaCluster
NAME         VERSION   DESIRED MEMBERS   READY MEMBERS   AGE
bar-pravega  0.4.0     7                 7               2m
```

```
$ kubectl get all -l pravega_cluster=bar-pravega
NAME                                          READY   STATUS    RESTARTS   AGE
pod/bar-pravega-controller-64ff87fc49-kqp9k   1/1     Running   0          2m
pod/bar-pravega-segmentstore-0                1/1     Running   0          2m
pod/bar-pravega-segmentstore-1                1/1     Running   0          1m
pod/bar-pravega-segmentstore-2                1/1     Running   0          30s

NAME                                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
service/bar-pravega-controller              ClusterIP   10.23.244.3   <none>        10080/TCP,9090/TCP   2m
service/bar-pravega-segmentstore-headless   ClusterIP   None          <none>        12345/TCP            2m

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/bar-pravega-controller-64ff87fc49       1         1         1       2m

NAME                                            DESIRED   CURRENT   AGE
statefulset.apps/bar-pravega-segmentstore       3         3         2m
```

By default, a `PravegaCluster` instance is only accessible within the cluster through the Controller `ClusterIP` service. From within the Kubernetes cluster, a client can connect to Pravega at:

```
tcp://[CLUSTER_NAME]-pravega-controller.[NAMESPACE]:9090
```

And the `REST` management interface is available at:

```
http://[CLUSTER_NAME]-pravega-controller.[NAMESPACE]:10080/
```

Check out the [external access documentation](doc/external-access.md) if your clients need to connect to Pravega from outside Kubernetes.

Check out the [exposing Segmentstore service on single IP address](https://github.com/pravega/pravega-operator/blob/4aa88641c3d5a1d5afbb2b9e628846639fd13290/doc/external-access.md#exposing-segmentstore-service-on-single-ip-address-and-different-ports) if your clients need to connect to Pravega Segment store on the same IP address from outside Kubernetes.

### Scale a Pravega cluster

You can scale Pravega components independently by modifying their corresponding field in the Pravega resource spec. You can either `kubectl edit` the cluster or `kubectl patch` it. If you edit it, update the number of replicas for Controller, and/or Segment Store and save the updated spec.

Example of patching the Pravega resource to scale the Segment Store instances to 4.

```
kubectl patch PravegaCluster [CLUSTER_NAME] --type='json' -p='[{"op": "replace", "path": "/spec/pravega/segmentStoreReplicas", "value": 4}]'
```

### Upgrade a Pravega cluster

Check out the [upgrade guide](doc/upgrade-cluster.md).

### Uninstall the Pravega cluster

```
$ helm uninstall [PRAVEGA_RELEASE_NAME]
$ kubectl delete -f ./example/pvc-tier2.yaml
```

### Uninstall the Operator

> Note that the Pravega clusters managed by the Pravega operator will NOT be deleted even if the operator is uninstalled.

```
$ helm uninstall [PRAVEGA_OPERATOR_RELEASE_NAME]
```

If you want to delete the Pravega clusters, make sure to do it before uninstalling the operator.

### Manual installation

You can also manually install/uninstall the operator and Pravega with `kubectl` commands. Check out the [manual installation](doc/manual-installation.md) document for instructions.

## Configuration

Check out the [configuration document](doc/configuration.md).

## Development

Check out the [development guide](doc/development.md).

## Releases  

The latest Pravega releases can be found on the [Github Release](https://github.com/pravega/pravega-operator/releases) project page.

## Troubleshooting

Check out the [troubleshooting document](doc/troubleshooting.md).
