# AWX Installation Methods – Technical Evaluation & Production Recommendation

[![AWX Version](https://img.shields.io/badge/AWX-24.x-blue)](https://github.com/ansible/awx)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-k3s%20v1.33.4-326CE5)](https://k3s.io/)
[![License](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)

---

## 📌 Purpose of This Document

This repository provides a **technical evaluation of multiple AWX installation methods**, based on real deployment tests.  
The objective is to **identify a reliable, maintainable, and production-ready installation strategy**.

The findings are intended to support **architectural and operational decisions**.

---

## 📑 Table of Contents

- [Executive Summary](#executive-summary)
- [Scope & Methodology](#scope--methodology)
- [Installation Methods Evaluated](#installation-methods-evaluated)
- [Comparative Summary](#comparative-summary)
- [Production Recommendation](#production-recommendation)
- [Reference Installation Guide](#reference-installation-guide)
- [Security & Production Considerations](#security--production-considerations)
- [Conclusion](#conclusion)

---

## 🎯 Executive Summary

Three AWX installation approaches were evaluated over **more than one week of hands-on testing**.

**Outcome:**

> **Only the Kubernetes-based deployment using the official AWX Operator is suitable for production use.**

All other methods are either **obsolete**, **unsupported**, or **introduce unacceptable operational and security risks**.

---

## 🔬 Scope & Methodology

- AWX versions tested: **17.x → 24.x**
- Focus criteria:
  - Official support status
  - Upgrade path & maintainability
  - Security posture
  - Operational stability
  - Time-to-production
- Environment:
  - Linux (RHEL-like)
  - k3s Kubernetes v1.33.4
  - Single-node (expandable to HA)

---

## 🧪 Installation Methods Evaluated

### Method 1 – Ansible Playbook (Legacy Installer)

```bash
ansible -i inventory install.yaml
````

| Aspect      | Result     |
| ----------- | ---------- |
| AWX Version | 17.0.1     |
| Status      | Functional |
| Support     | Deprecated |
| Security    | Outdated   |

**Observations**

* Installation completes successfully
* Minimal configuration changes required
* Stable runtime behavior

**Limitations**

* Last supported AWX version dates back to **2019**
* No security updates
* Obsolete dependency stack (Python, Django, PostgreSQL)
* No upgrade path beyond AWX 17.x

**Assessment**

> ⚠ **Not suitable for production**
> Functional, but technically obsolete and non-compliant with modern security standards.

---

### Method 2 – Docker Compose (Deprecated)

```bash
make docker-compose-build
make docker-compose
```

| Aspect          | Result     |
| --------------- | ---------- |
| AWX Version     | 24.6.1     |
| Support         | Removed    |
| Stability       | Unreliable |
| Maintainability | Poor       |

**Key Findings**

* Requires **direct modification of AWX source code**
* Multiple dependency conflicts (Django, OpenSSL, sqlparse)
* Database migrations require unsafe manual fixes

**Operational Risks**

* No official support
* Broken upgrade path
* High risk of runtime failures
* Divergence from upstream codebase

**Assessment**

> 🚫 **Explicitly not recommended**
> This method is deprecated and unsuitable for any production environment.

---

### Method 3 – Kubernetes (k3s) + AWX Operator

```bash
kubectl apply -k .
```

| Aspect         | Result       |
| -------------- | ------------ |
| AWX Version    | 24.x         |
| Support        | Official     |
| Stability      | High         |
| Time to Deploy | < 20 minutes |

**Advantages**

* Officially supported installation method
* No source code modifications
* Operator-managed lifecycle (deploy, upgrade, rollback)
* Compatible with:

  * k3s
  * Standard Kubernetes
  * OpenShift

**Assessment**

> ✅ **Production-ready and recommended**

---

## ⚡ Comparative Summary

| Method                    | Supported | Secure | Maintainable | Production |
| ------------------------- | --------- | ------ | ------------ | ---------- |
| Ansible Playbook          | ❌         | ❌      | ❌            | ❌          |
| Docker Compose            | ❌         | ❌      | ❌            | ❌          |
| **Kubernetes + Operator** | ✅         | ✅      | ✅            | ✅          |

---

## 🚀 Production Recommendation

### ✔ Recommended Architecture

* **AWX deployed via AWX Operator**
* **Kubernetes-based platform**

  * k3s for lightweight or edge use cases
  * Kubernetes / OpenShift for HA and enterprise environments

### ❌ Explicitly Not Recommended

* Legacy Ansible installer
* Docker Compose-based deployment
* Any installation requiring AWX source code modification

---

## 🛠 Reference Installation Guide (k3s)

### 1. Install k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.4+k3s1 sh -
```

### 2. Verify Cluster

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

### 3. Deploy AWX Operator & Instance

```bash
kubectl create namespace awx
kubectl apply -k .
```

### 4. Retrieve Admin Credentials

```bash
kubectl get secret awx-admin-password \
  -n awx -o jsonpath="{.data.password}" | base64 --decode
```

---

## 🔐 Security & Production Considerations

* Change default admin credentials immediately
* Use HTTPS (Ingress + cert-manager)
* Configure persistent storage for PostgreSQL
* Implement backup & restore strategy
* Restrict network access via firewall / network policies
* Regularly update AWX Operator and images

---

## 🧾 Conclusion

This evaluation demonstrates that:

* **Legacy and deprecated installation methods introduce unacceptable risk**
* **Kubernetes + AWX Operator is the only sustainable choice**
* k3s provides an excellent balance between **simplicity and production readiness**

> **Recommendation:**
> Adopt AWX Operator on Kubernetes as the standard deployment model.

---

## 📄 Metadata

* **AWX Version:** 24.x
* **AWX Operator:** 2.19.1
* **Kubernetes:** k3s v1.33.4
* **Last Review:** January 2026

---

## ✍ Author

**OUAZENE Bilal**

---

> *Design for upgradeability, security, and long-term maintenance — not just installation success.*
