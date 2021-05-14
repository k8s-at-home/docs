# Introduction

Running applications in Kubernetes is a bit different that in a regular Docker
Swarm or Docker Compose setup. Our
[container images](https://github.com/k8s-at-home/container-images) are
tailored for running in Kubernetes.

## Purpose

The goal of this project and the container images are to support
[Semantic Versioning](https://semver.org/) and multiple architectures.

We try to keep a [KISS principle](https://en.wikipedia.org/wiki/KISS_principle)
when building these images, which means no
[s6-overlay](https://github.com/just-containers/s6-overlay),
are all built on top of [ubuntu:focal](https://hub.docker.com/_/ubuntu), and are 
only amd64 and arm64 architectures.

## Deprecations

Container images that were once here but now are gone it is likely that the
application...

- developers are supporting their own images and conforming to
[Semantic Versioning](https://semver.org/), or
- has been replaced with a different application, or
- maintenance cost it too high to continue to build images for, or
- is no longer maintained
