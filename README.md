# Introduction to GitOps

This repository contains instructions for setting up FluxCD and templates for implementing a (simplistic) GitOps workflow, allowing for continuous deployment / delivery. The demo application may be simple, but the principles can be applied in any Kubernetes environment.

## What is GitOps?

GitOps is based on the foundation "git as the single source of truth". All code running on a cluster is persisted in git in a declarative way.

## What is a Kustomization?

A Kustomization in the Kubernetes sense (there are also Kustimizations in FluxCD, more on that below) allows for enforcing the DRY principle across clusters. You can organize your repository with so-called overlays and let the Kustomization render the differences.

The idea behind Kustomizations is that you have a so-called base and only need to define what differs between the environments that you wish to control, e.g. dev, staging and production, or blue-green deployments.

## What is FluxCD?

FluxCD is a set of controllers that allow developers to deploy code to Kubernetes the GitOps way. It consists of controllers that serve different purposes:

- Source Controller - used for fetching manifests from a source, e.g. a git repository
- Kustomize Controller - used for compiling the manifests from a source
- Helm Controller - used for deploying helm manifests, similar to Kustomizations

# Live Demo

Local dev only, requires a git repository that is reachable through either ssh or https. This demo uses Docker and minikube.

Follow the installation guides here:

* [Docker](https://docs.docker.com/desktop/)
* [Minikube](https://minikube.sigs.k8s.io/docs/start/)

## Agenda

1.  [Running Gitea](#running-gitea)
2.  [Create Flux Repository](#create-flux-repository-in-gitea)
3.  [Start minikube - Local Kubernetes Cluster](#start-minikube---local-kubernetes-cluster)
4.  [Bootstrap FluxCD](#bootstrap-fluxcd)
5.  [Create Namespaces](#create-namespaces)
6.  [Install Traefik Helm Chart](#install-treafik-helm-chart)
7.  [Kubernetes Kustomizations](#kubernetes-kustomizations)
8.  [Adding External git Repositories to Flux](#adding-external-git-repositories-to-flux)
9.  [Adding Overlays for the Other Environments](#adding-overlays-for-the-other-environments)
10. [Flux Reconciliation](#flux-reconciliation)
