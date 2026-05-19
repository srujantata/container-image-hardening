# Container Image Hardening

[![CI](https://github.com/srujantata/container-image-hardening/actions/workflows/scan.yml/badge.svg)](https://github.com/srujantata/container-image-hardening/actions)

A practical toolkit for building and auditing secure container images:
**Hadolint** linting, **multi-stage distroless** builds, **Trivy** CVE scanning,
**kube-bench** CIS benchmarking, and **Cosign** keyless signing — all wired into CI.

---

## What's Covered

| Technique | Tool | Outcome |
|-----------|------|---------|
| Dockerfile best-practice enforcement | Hadolint | Catches misuse at write-time |
| Minimal attack surface | Multi-stage + distroless base | No shell, no package manager in final image |
| Known CVE gating | Trivy | Blocks CI on CRITICAL/HIGH |
| CIS Kubernetes Benchmark | kube-bench | Node and control-plane posture report |
| Tamper-proof image provenance | Cosign (keyless) | Signature stored in registry, verifiable anywhere |

---

## Architecture

```
Developer writes Dockerfile
        │
        ▼
  Hadolint (lint)  ──► violations? ──► PR blocked
        │ clean
        ▼
  docker build (multi-stage)
  └── Stage 1: golang:1.22 (build)
  └── Stage 2: gcr.io/distroless/base (final — no shell)
        │
        ▼
  Trivy image scan  ──► CRITICAL CVE? ──► PR blocked
        │ clean
        ▼
  Push to GHCR
        │
        ▼
  Cosign sign (keyless OIDC — no stored key)
        │
        ▼
  Deploy to cluster
        │
        ▼
  kube-bench Job  ──► CIS compliance report in CI output
```

---

## Hadolint

Hadolint parses your Dockerfile and flags violations against Docker best practices.
Run it in CI and locally before committing.

```bash
# Install
brew install hadolint   # macOS
choco install hadolint  # Windows

# Run
hadolint Dockerfile
```

Example violations:
```
Dockerfile:3 DL3008 Pin versions in apt-get install: apt-get install -y curl
Dockerfile:7 DL3006 Always tag the version of the image you're using
Dockerfile:11 SC2086 Double quote to prevent globbing: COPY $SRC /app
```

Configuration (`.hadolint.yaml`):
```yaml
ignore:
  - DL3018   # accept unpinned apk in build stage only
failure-threshold: error
```

---

## Multi-Stage Distroless Build

Stage 1 does all compilation in a full SDK image.
Stage 2 copies only the binary into a distroless image — no shell, no apt, no libc extras.

```dockerfile
# Dockerfile
# ── Stage 1: Build ────────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# ── Stage 2: Runtime (distroless — no shell, no package manager) ──────────────
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

Before vs after:
```
Base image: golang:1.22       → 819 MB
Base image: distroless/static → 2.5 MB   (-99.7%)

CVEs (Trivy):
  golang:1.22        → 23 MEDIUM, 6 HIGH, 2 CRITICAL
  distroless/static  → 0 CVEs
```

---

## Trivy Scanning in CI

```yaml
# .github/workflows/scan.yml (excerpt)
- name: Run Trivy vulnerability scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/srujantata/my-app:${{ github.sha }}
    format: table
    exit-code: 1           # fail CI if vulnerabilities found
    severity: CRITICAL,HIGH
    ignore-unfixed: true   # skip CVEs with no upstream fix yet
```

Example Trivy output:
```
ghcr.io/srujantata/my-app:abc1234 (debian 12.5)

OS/Arch: linux/amd64
Total: 0 (CRITICAL: 0, HIGH: 0)
```

If vulnerabilities are found:
```
Total: 2 (CRITICAL: 1, HIGH: 1)

┌──────────────────┬────────────────┬──────────┬──────────┬──────────────────────┐
│     Library      │ Vulnerability  │ Severity │ Installed│ Fixed Version        │
├──────────────────┼────────────────┼──────────┼──────────┼──────────────────────┤
│ libssl3          │ CVE-2024-5535  │ CRITICAL │ 3.0.13-1 │ 3.0.14-1~deb12u1     │
│ curl             │ CVE-2024-2398  │ HIGH     │ 7.88.1-1 │ 7.88.1-10+deb12u6    │
└──────────────────┴────────────────┴──────────┴──────────┴──────────────────────┘
```

---

## kube-bench CIS Benchmark

kube-bench runs the CIS Kubernetes Benchmark checks against control-plane and worker nodes.

```yaml
# k8s/kube-bench-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: kube-system
spec:
  template:
    spec:
      hostPID: true
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:v0.7.3
        command: ["kube-bench", "run", "--targets", "node"]
        volumeMounts:
        - name: var-lib-kubelet
          mountPath: /var/lib/kubelet
          readOnly: true
        - name: etc-systemd
          mountPath: /etc/systemd
          readOnly: true
      restartPolicy: Never
      volumes:
      - name: var-lib-kubelet
        hostPath:
          path: /var/lib/kubelet
      - name: etc-systemd
        hostPath:
          path: /etc/systemd
```

Sample output:
```
[INFO] 4 Worker Node Security Configuration
[INFO] 4.1 Worker Node Configuration Files
[PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 644
[PASS] 4.1.2 Ensure that the kubelet service file ownership is set to root:root
[WARN] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true
[FAIL] 4.2.11 Ensure that the RotateKubeletServerCertificate argument is set to true

== Summary node ==
16 checks PASS
1 checks FAIL
3 checks WARN
```

---

## Cosign Keyless Signing

```yaml
# .github/workflows/sign.yml (excerpt)
- name: Sign image with Cosign
  env:
    COSIGN_EXPERIMENTAL: "true"
  run: |
    cosign sign --yes \
      ghcr.io/srujantata/my-app@${{ steps.build.outputs.digest }}
```

```bash
# Verify from anywhere
cosign verify \
  --certificate-identity-regexp="https://github.com/srujantata/container-image-hardening" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/srujantata/my-app:latest
```

---

## Skills Demonstrated

- Dockerfile linting with Hadolint and best-practice enforcement
- Multi-stage builds reducing image size and CVE surface by 99%+
- Distroless base images (no shell, no package manager, non-root by default)
- Trivy vulnerability scanning with severity-gated CI
- kube-bench CIS Kubernetes Benchmark node auditing
- Cosign keyless image signing using GitHub Actions OIDC
