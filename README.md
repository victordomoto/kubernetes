# Kubernetes

This repository is dedicated to studying Kubernetes and preparing for the Certified Kubernetes Administrator (CKA) exam.

The content is organized according to the main CKA competency domains, making it easier to focus on specific topics and track learning progress.

# Directory Structure

This repository contains five directories, organized according to the CKA competency domains.

## storage: 10%

* Implement StorageClasses and dynamic volume provisioning
* Configure volume types, access modes, and reclaim policies
* Manage PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs)

## troubleshooting: 30%

* Troubleshoot clusters and nodes
* Troubleshoot cluster components
* Monitor cluster and application resource usage
* Manage and evaluate container output streams
* Troubleshoot services and networking

## workloads-scheduling: 15%

* Understand application deployments and perform rolling updates and rollbacks
* Use ConfigMaps and Secrets to configure applications
* Configure workload autoscaling
* Understand the primitives used to create robust, self-healing application deployments
* Configure Pod admission and scheduling (resource limits, node affinity, taints, tolerations, etc.)

## cluster-architecture: 25%

* Manage Role-Based Access Control (RBAC)
* Prepare the underlying infrastructure for Kubernetes cluster installation
* Create and manage Kubernetes clusters using kubeadm
* Manage the Kubernetes cluster lifecycle
* Implement and configure a highly available control plane
* Use Helm and Kustomize to install and manage cluster components
* Understand extension interfaces (CNI, CSI, CRI, etc.)
* Understand Custom Resource Definitions (CRDs), and install and configure operators

## services-networking 20%

* Understand connectivity between Pods
* Define and enforce NetworkPolicies
* Use ClusterIP, NodePort, and LoadBalancer Service types and endpoints
* Use the Gateway API to manage ingress traffic
* Understand how to use Ingress controllers and Ingress resources
* Understand and use CoreDNS
