# AWX Installation Methods Comparison

[![AWX Version](https://img.shields.io/badge/AWX-24.x-blue)](https://github.com/ansible/awx)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-k3s%20v1.33.4-326CE5)](https://k3s.io/)
[![License](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)

## 📋 Table of Contents

- [Executive Summary](#executive-summary)
- [Methods Tested](#methods-tested)
- [Quick Comparison](#quick-comparison)
- [Recommended Installation](#recommended-installation)
- [Detailed Analysis](#detailed-analysis)
- [Conclusion](#conclusion)

## 🎯 Executive Summary

This repository documents extensive testing of three AWX installation methods. After **7-8 days of testing**, only one method is viable for production use.

**TL;DR**: Use Kubernetes + AWX Operator. Skip everything else.

## 🧪 Methods Tested

### ❌ Method 1: Ansible Playbook (OBSOLETE)

```bash
ansible -i inventory install.yaml
```

| Status | Time Spent | Result |
|--------|------------|--------|
| ❌ Obsolete | 2 days | Only works on AWX < 18.x (2019) |

**Problems:**
- Only compatible with AWX 17.0.1 (released in 2019)
- Critical security vulnerabilities
- No security patches since 2019
- Completely abandoned by the project

**Verdict:** 🚫 **NEVER USE** - Security nightmare

---

### ⚠️ Method 2: Docker Compose (NOT RECOMMENDED)

```bash
make docker-compose-build
make docker-compose
```

| Status | Time Spent | Modifications Required |
|--------|------------|------------------------|
| ⚠️ Unsupported | 5-6 days | 5 source files |

**Required Code Modifications:**

1. **`tools/docker-compose/inventory`**
   ```ini
   admin_password="awxpass123"
   pg_password="awxpass123"
   broadcast_websocket_secret="awxpass123"
   secret_key="awxpass123"
   ```

2. **`requirements/requirements.in`**
   ```python
   # BEFORE: django==4.2.10
   # AFTER:
   django==4.2.26
   sqlparse>=0.5.2
   ```

3. **`requirements/requirements.txt`**
   - Update sqlparse dependency

4. **`Dockerfile.j2`**
   ```dockerfile
   # BEFORE: openssl-3.0.7
   # AFTER: openssl
   ```

5. **`awx/main/migrations/_dab_rbac.py`**
   - Critical database migration fix (⚠️ dangerous)

**Problems:**
- ❌ 5 source code modifications required
- ❌ Unstable with broken features
- ❌ Non-standard database schema
- ❌ Impossible to update
- ❌ No community support
- ❌ Nearly 1 week of debugging

**Verdict:** 🚫 **AVOID** - Unmaintainable, unreliable

---

### ✅ Method 3: Kubernetes k3s + AWX Operator (RECOMMENDED)

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.4+k3s1 sh -
kubectl apply -k .
```

| Status | Time Required | Modifications |
|--------|---------------|---------------|
| ✅ Official | **15-20 minutes** | **ZERO** |

**Why k3s?**
- 💪 Lightweight: < 512 MB RAM (vs 2-4 GB for k8s, 8+ GB for OpenShift)
- 🎯 Production-ready Kubernetes distribution
- 🔄 Works on standard Kubernetes and OpenShift too
- 🚀 Perfect for labs, edge computing, limited resources

**Features:**
- ✅ Zero code modifications
- ✅ Official Ansible/Red Hat support
- ✅ Automatic updates via operator
- ✅ All features working
- ✅ Production-ready from day 1

**Verdict:** ✅ **RECOMMENDED** - Only viable method

## ⚡ Quick Comparison

| Method | Setup | Debug Time | Total Time | Status |
|--------|-------|------------|------------|--------|
| Ansible Playbook | 30 min | 2 days | **2 days** | ❌ Obsolete |
| Docker Compose | 30 min | 5-6 days | **~1 week** | ⚠️ Broken |
| **k3s + Operator** | **20 min** | **0** | **20 min** | ✅ **Works** |

### Time Wasted on Deprecated Methods

```
┌─────────────────────────────────────────┐
│ Ansible Playbook:    ████████ (2 days)  │
│ Docker Compose:      ████████████████    │
│                      ████████ (6 days)   │
│ ────────────────────────────────────────│
│ Total Wasted:        8 DAYS             │
│                                          │
│ k3s + Operator:      ▌ (20 minutes)     │
└─────────────────────────────────────────┘
```

## 🚀 Recommended Installation

### Prerequisites

```bash
# Disable firewall (recommended for lab/test)
systemctl disable firewalld --now

# OR configure firewall for production
firewall-cmd --permanent --add-port=6443/tcp    # Kubernetes API
firewall-cmd --permanent --add-port=30080/tcp   # AWX Web UI
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16  # Pods
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16  # Services
firewall-cmd --reload
```

### Step-by-Step Installation

#### 1️⃣ Install k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.4+k3s1 sh -
```

⏱️ **Time:** 2-3 minutes

#### 2️⃣ Verify k3s Installation

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl get storageclass
```

Expected output:
- Node status: `Ready`
- All system pods: `Running`
- StorageClass: `local-path` available

#### 3️⃣ Create AWX Namespace

```bash
kubectl create namespace awx
```

#### 4️⃣ Create AWX Configuration

Create `awx-instance.yaml`:

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
  nodeport_port: 30080
```

#### 5️⃣ Create Kustomization File

Create `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
  - awx-instance.yaml
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
namespace: awx
```

#### 6️⃣ Deploy AWX

```bash
kubectl apply -k .
```

⏱️ **Time:** 5-10 minutes

#### 7️⃣ Monitor Deployment

```bash
watch -n 30 'kubectl get pods -n awx'
```

Expected pods:
- `awx-operator-controller-manager-*`
- `awx-postgres-*`
- `awx-web-*`
- `awx-task-*`

#### 8️⃣ Get Admin Password

```bash
kubectl get secret awx-admin-password \
  -o jsonpath="{.data.password}" -n awx | base64 --decode
echo
```

#### 9️⃣ Access AWX

```
URL: http://YOUR_SERVER_IP:30080
Username: admin
Password: (from step 8)
```

### 📊 Installation Timeline

```
Total Time: 15-20 minutes

├─ Configure firewall      (1 min)
├─ Install k3s            (2-3 min)
├─ Verify k3s             (1 min)
├─ Create config files    (2 min)
├─ Deploy AWX             (5-10 min)
└─ Access & verify        (2 min)
```

## 📖 Detailed Analysis

### Why Ansible Playbook Failed

- 🔴 Only supports AWX < 18.x
- 🔴 Successfully installed AWX 17.0.1 (2019)
- 🔴 5+ years of unpatched security vulnerabilities
- 🔴 Obsolete dependencies (Django, Python, PostgreSQL)
- 🔴 No updates or support available
- 🔴 **2 days wasted** on an obsolete method

### Why Docker Compose Failed

**5 Critical Files Modified:**

| File | Issue | Risk Level |
|------|-------|------------|
| `inventory` | Auto-generated secrets fail | Medium |
| `requirements.in` | Django 4.2.10 incompatible with Python 3.11+ | High |
| `requirements.txt` | sqlparse dependency outdated | Medium |
| `Dockerfile.j2` | OpenSSL 3.0.7 not in repos | Medium |
| `_dab_rbac.py` | Database migration crashes | **CRITICAL** |

**Result After 1 Week:**
- ✅ Installation successful
- ❌ Unstable features
- ❌ Random task failures
- ❌ Unreliable inventory sync
- ❌ Broken notifications
- ❌ Degraded performance
- ❌ Impossible to update
- ❌ **Nearly 1 week wasted**

### Why Kubernetes Works

**Zero modifications required:**
- ✅ Official installation method
- ✅ Works in 20 minutes
- ✅ No debugging needed
- ✅ Automatic updates
- ✅ Full feature support
- ✅ Production-ready
- ✅ Active community support

**Kubernetes Compatibility:**

| Platform | RAM Required | Status |
|----------|--------------|--------|
| k3s | 512 MB+ | ✅ Tested & Recommended |
| Kubernetes | 4 GB+ | ✅ Supported |
| OpenShift | 8 GB+ | ✅ Supported |
| k0s, microk8s | 512 MB+ | ✅ Compatible |

## 🎯 Final Recommendation

### ✅ DO THIS

```bash
# Install AWX on k3s (20 minutes)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.4+k3s1 sh -
kubectl create namespace awx
kubectl apply -k .
```

### 🚫 DON'T DO THIS

```bash
# ❌ Ansible Playbook - OBSOLETE
ansible -i inventory install.yaml

# ❌ Docker Compose - BROKEN
make docker-compose-build && make docker-compose
```

## 📊 Key Metrics

### Time Investment Summary

```
┌──────────────────────────────────────────────┐
│  WASTED TIME: 7-8 DAYS                       │
│  ├─ Ansible Playbook:     2 days (obsolete)  │
│  └─ Docker Compose:       6 days (broken)    │
│                                               │
│  PRODUCTIVE TIME: 20 MINUTES                 │
│  └─ k3s + AWX Operator:   20 min (works!)    │
└──────────────────────────────────────────────┘
```

### Success Rate

| Method | Success | Production Ready | Maintainable |
|--------|---------|------------------|--------------|
| Ansible Playbook | ❌ | ❌ | ❌ |
| Docker Compose | ⚠️ Partial | ❌ | ❌ |
| k3s + Operator | ✅ | ✅ | ✅ |

## 🔧 Troubleshooting

### Common k3s Issues

**Issue: Pods stuck in Pending**
```bash
kubectl describe pod <pod-name> -n awx
kubectl get events -n awx
```

**Issue: Can't access AWX on port 30080**
```bash
# Check service
kubectl get svc -n awx

# Check firewall
firewall-cmd --list-ports
```

**Issue: Forgot admin password**
```bash
kubectl get secret awx-admin-password \
  -o jsonpath="{.data.password}" -n awx | base64 --decode
```

## 📚 Additional Resources

- [Official AWX Documentation](https://ansible.readthedocs.io/projects/awx/en/latest/)
- [AWX Operator GitHub](https://github.com/ansible/awx-operator)
- [k3s Documentation](https://docs.k3s.io/)
- [Ansible Documentation](https://docs.ansible.com/)

## 📝 Notes

### Security Considerations

- 🔒 Change default admin password immediately
- 🔒 Use HTTPS in production (Ingress + cert-manager)
- 🔒 Regular backups of AWX database
- 🔒 Keep AWX Operator updated

### Production Recommendations

- Use Kubernetes standard or OpenShift for high availability
- Configure persistent storage for PostgreSQL
- Set up proper monitoring (Prometheus + Grafana)
- Implement backup strategy
- Configure RBAC properly

## 🤝 Contributing

This is a technical report documenting real-world testing. If you found alternative methods or improvements, please share your experience.

## 📄 License

Apache 2.0

## ✍️ Author

**OUAZENE Bilal**  
AXA Assurances  
January 2026

---

### 💡 Key Takeaway

> **Save yourself 8 days of frustration:**  
> Skip Ansible Playbook and Docker Compose.  
> Go straight to Kubernetes + AWX Operator.  
> **It just works.™**

---

**Last Updated:** January 2026  
**AWX Version Tested:** 24.x  
**k3s Version:** v1.33.4+k3s1  
**AWX Operator Version:** 2.19.1
