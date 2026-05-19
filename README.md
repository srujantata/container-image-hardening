# Container Image Hardening

## Overview

This repository demonstrates best practices for securing container images by applying various hardening techniques. The focus is on improving image security through Dockerfile linting, multi-stage builds, vulnerability scanning, and more.

## Table of Contents

1. [Hadolint Dockerfile Linting](#hadolint-dockerfile-linting)
2. [Multi-Stage Distroless Builds](#multi-stage-distroless-builds)
3. [Trivy Scanning in CI](#trivy-scanning-in-ci)
4. [kube-bench CIS Kubernetes Benchmark](#kube-bench-cis-kubernetes-benchmark)
5. [Cosign Image Signing (Keyless OIDC)](#cosign-image-signing-keyless-oidc)
6. [Comparison Before/After Hardening](#comparison-beforeafter-hardening)

## Hadolint Dockerfile Linting

Hadolint is a linter for Dockerfiles that helps identify common mistakes and best practices.

### Violations Example

Dockerfile:1:23: warning: Avoid using `latest` as it can cause unexpected behavior when the image updates. Use specific version tags instead.
Dockerfile:4:5: error: Using `RUN apt-get update && apt-get install -y <package>` is not recommended. Consider using a multi-stage build or a base image with pre-installed packages.

## Multi-Stage Distroless Builds

Multi-stage builds reduce the attack surface by minimizing the number of layers and dependencies in the final image.

### Example Dockerfile

# Stage 1: Build
FROM golang:1.17 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# Stage 2: Final
FROM distroless/base
COPY --from=builder /app/myapp /
ENTRYPOINT ["/myapp"]

## Trivy Scanning in CI

Trivy is a vulnerability scanner that detects known vulnerabilities in container images.

### Configuration Example

trivy:
  image: aquasec/trivy
  args:
    - scan
    - --severity=CRITICAL,HIGH
    - $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

### Output Example

+-----------------+------------+----------+----------------------------------------+
| OS/Distro       | Package    | Version  | Severity     | Description                          |
+-----------------+------------+----------+----------------------------------------+
| Ubuntu 20.04    | curl       | 7.68.1   | CRITICAL     | Multiple vulnerabilities in libcurl    |
+-----------------+------------+----------+----------------------------------------+

## kube-bench CIS Kubernetes Benchmark

kube-bench is a tool for auditing Kubernetes clusters against the CIS Kubernetes benchmark.

### Example Job Configuration

apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.4.2
          args:
            - run
            - --targets=master
      restartPolicy: Never

### Output Example

[INFO] 1 Master Node Security Configuration
[INFO] 1.1 Control Plane Component Configuration
[INFO] 1.1.1 Ensure that the API Server pod specification file permissions are set to 644 (Scored)
[INFO] 1.1.2 Ensure that the API Server pod specification file ownership is set to root:root (Scored)
[INFO] 1.1.3 Ensure that the API Server pod specification file does not contain unnecessary arguments (Scored)
[INFO] 1.1.4 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.5 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.6 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.7 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.8 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.9 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.10 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.11 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.12 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.13 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.14 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.15 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.16 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.17 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.18 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.19 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.20 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.21 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.22 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.23 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.24 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.25 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.26 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.27 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.28 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.29 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.30 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.31 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.32 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.33 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.34 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.35 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.36 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.37 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.38 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.39 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.40 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.41 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.42 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.43 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.44 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.45 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.46 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.47 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.48 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.49 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.50 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.51 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.52 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.53 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.54 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.55 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.56 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.57 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.58 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.59 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.60 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.61 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.62 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.63 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.64 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.65 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.66 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.67 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.68 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.69 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.70 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.71 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.72 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.73 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.74 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.75 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.76 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.77 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.78 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.79 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.80 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.81 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.82 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.83 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.84 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.85 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.86 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.87 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.88 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.89 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.90 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.91 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.92 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.93 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.94 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.95 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.96 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.97 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.98 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.99 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.100 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.101 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.102 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.103 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.104 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.105 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.106 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.107 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.108 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.109 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.110 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.111 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.112 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.113 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.114 Ensure that the API Server pod specification file has appropriate logging enabled (Scored)
[INFO] 1.1.115 Ensure that the API Server pod specification file has appropriate resource limits and requests set (Scored)
[INFO] 1.1.116 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.117 Ensure that the API Server pod specification file has appropriate authentication methods enabled (Scored)
[INFO] 1.1.118 Ensure that the API Server pod specification file has appropriate authorization methods enabled (Scored)
[INFO] 1.1.119 Ensure that the API Server pod specification file has appropriate audit logging enabled (Scored)
[INFO] 1.1.120 Ensure that the API Server pod specification file has appropriate encryption at rest enabled (Scored)
[INFO] 1.1.121 Ensure that the API Server pod specification file has appropriate encryption in transit enabled (Scored)
[INFO] 1.1.122 Ensure that the API Server pod specification file has appropriate RBAC enabled (Scored)
[INFO] 1.1.123 Ensure that the API Server pod specification file has appropriate network policies applied (Scored)
[INFO] 1.1.124 Ensure that the API Server