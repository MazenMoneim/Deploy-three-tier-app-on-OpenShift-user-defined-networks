<div align="center">

#  Deploy a Three-Tier App on OpenShift using User-Defined Networks

**Prime Series and Movies** components: Nginx **frontend**, Python API **backend** and MariaDB **database**,<br/>
isolated with **ClusterUserDefinedNetworks** and a **NetworkPolicy**, packaged as a **Helm chart**<br/>
and delivered by **Argo CD** from **GitLab**.

![OpenShift](https://img.shields.io/badge/OpenShift-EE0000?logo=redhatopenshift&logoColor=white)
![OVN-Kubernetes](https://img.shields.io/badge/OVN--Kubernetes-UDN%20%2F%20CUDN-2F6FEB)
![Helm](https://img.shields.io/badge/Helm-3-0F1689?logo=helm&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-OpenShift%20GitOps-EF7B4D?logo=argo&logoColor=white)
![GitLab](https://img.shields.io/badge/GitLab-source%20of%20truth-FC6D26?logo=gitlab&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?logo=mariadb&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-frontend-009639?logo=nginx&logoColor=white)

</div>

---

## Components

| Tier | Namespace | Runs | Networks |
|---|---|---|---|
| **Frontend** | `webapp-frontend` | Nginx | `default network` (primary) and `frontend-backend-net` |
| **Backend** | `webapp-backend` | Python API | `webapp-cudn`     (primary) and `frontend-backend-net` |
| **Database** | `webapp-database` | MariaDB StatefulSet  | `webapp-cudn`    (primary) |

---

## Contents

| Step | Section |
|:-:|---|
| 1 | [Architecture](#1-architecture) |
| 2 | [Repository layout](#2-repository-layout) |
| 3 | [Prepare the environment](#3-prepare-the-environment) |
| 4 | [The networks and how they are created](#4-the-networks-and-how-they-are-created) |
| 5 | [Put the code in GitLab](#5-put-the-code-in-gitlab) |
| 6 | [Verify](#6-verify) |
| 7 | [Day-2 operations](#7-day-2-operations) |
| 8 | [Troubleshooting](#8-troubleshooting) |
| 9 | [Application UI](#9-application-ui) |

---

## 1. Architecture

###  Network topology

<img width="2589" height="3056" alt="CamScanner 13-10-2026 15 54" src="https://github.com/user-attachments/assets/40ff7369-02c4-47ad-8d47-317611849a57" />


```mermaid

```

Each tier touches only the networks beside it. The frontend and the
database never share a network, so the database can only be reached through the backend.

| Network | Type | Subnet | Attached to | Purpose |
|---|---|---|---|---|
|  Cluster default network | OVN-Kubernetes default | cluster pod CIDR | `webapp-frontend` | Route traffic from the router to the frontend |
|  `webapp-cudn` | **Primary** CUDN · Layer2  | `10.50.0.0/16` | namespaces labelled `app-tier=webapp` | Isolated network for backend ↔ database |
|  `frontend-backend-net` | **Secondary** CUDN · Layer2 | `192.168.50.0/24` | `webapp-frontend`, `webapp-backend` | The only path from frontend to backend |

###  Interfaces inside each pod

| Pod | Interface | Network | Address |
|---|---|---|---|
| `frontend-web` | `eth0` | `Default network` | cluster pod CIDR |
| `frontend-web` | `net1` | `frontend-backend-net` | `192.168.50.x` |
| `backend-api` | `ovn-udn1` | `webapp-cudn` (primary) | `10.50.x.x` |
| `backend-api` | `net1` | `frontend-backend-net` | `192.168.50.y` |
| `webapp-db-0` | `ovn-udn1` | `webapp-cudn` (primary) | `10.50.x.x` |

###  Who can reach what

| From | To | | Why |
|---|---|:-:|---|
| Browser | frontend | ✅ | Route on the default network |
| Browser | backend | ❌ | No Route; backend lives on `webapp-cudn` |
| Frontend | backend `192.168.50.y:8080` | ✅ | Shared secondary network `frontend-backend-net` |
| Frontend | backend Service or `10.50.x.x` | ❌ | Frontend is not on `webapp-cudn` |
| Frontend | database | ❌ | Frontend is not on `webapp-cudn` |
| Backend (`app=backend-api`) | database `:3306` | ✅ | Same primary network, allowed by the NetworkPolicy |
| Any other pod on `webapp-cudn` | database | ❌ | Blocked by NetworkPolicy `restrict-db-access` |

###  A request, end to end

| Step | From | To | Network |
|:-:|---|---|---|
| 1 | Browser | Route, then nginx in `frontend-web` | Default network |
| 2 | nginx | `backend-api` on `192.168.50.y:8080` | `frontend-backend-net` |
| 3 | `backend-api` | MariaDB on port `3306` | `webapp-cudn` |




---

## 2. Repository layout

```
.
├── README.md                    this guide
├── argocd/
│   ├── application.yaml         Argo CD Application (Helm source)
│   ├── rbac.yaml                lets Argo CD manage the CUDNs
│   └── repository.yaml          template for the GitLab repository Secret
├── charts/prime-app/
│   ├── Chart.yaml
│   ├── values.yaml              images, namespaces, networks, credentials, route
│   ├── files/
│   │   ├── frontend/            index.html, style.css, app.js, art.js, config.js, nginx.conf, upstream_sync.py
│   │   ├── backend/             app.py, PyMySQL wheel
│   │   └── database/            10-schema-and-seed.sh, 99-webapp.cnf
│   └── templates/
│       ├── platform/            namespaces + the two CUDNs
│       ├── database/            StatefulSet, Services, Secret, ConfigMaps, NetworkPolicy
│       ├── backend/             Deployment, Service, Secret, ConfigMaps
│       └── frontend/            Deployment + sidecar, RBAC, Service, Route, ConfigMaps
└── scripts/
    └── verify-tiers.sh          checks every tier and network path (PASS/FAIL)
```

---

## 3. Prepare the environment


### 3.1 Check the cluster

```bash
oc version
oc get network.config cluster -o jsonpath='{.status.networkType}{"\n"}'     # OVNKubernetes
oc get clusteruserdefinednetworks.k8s.ovn.org                               # CUDN present
oc get nodes
```

### 3.2 Check storage

The database needs a `ReadWriteOnce` volume.

```bash
oc get storageclass          # one class should be marked (default)
```



### 3.3 Check installing OpenShift GitOps and ArgoCD

```bash
oc get argocd -n openshift-gitops
```



### 3.5 Let your OpenShift user see applications in the Argo CD UI

```bash
oc adm groups new cluster-admins 
oc adm groups add-users cluster-admins "$(oc whoami)"
oc get argocd openshift-gitops -n openshift-gitops -o jsonpath='{.spec.rbac.policy}{"\n"}'
# should contain: g, cluster-admins, role:admin
```

```bash
# Argo CD UI - log in with "OpenShift"
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='https://{.spec.host}{"\n"}'
```

> [!NOTE]
> The default instance only shows applications to the `cluster-admins` group. Log out of
> Argo CD and back in after changing groups.

---

## 4. The networks and how they are created

###  Creation order (Argo CD sync waves)

<img width="710" height="111" alt="image" src="https://github.com/user-attachments/assets/86d272ed-2958-40a0-8183-c4329837da11" />



###  What happens when a CUDN is created

| Step | What happens |
|:-:|---|
| 1 | Argo CD creates the namespaces with the `app-tier=webapp` and primary-UDN labels |
| 2 | Argo CD creates the two ClusterUserDefinedNetworks |
| 3 | OVN-Kubernetes creates a NetworkAttachmentDefinition in each matching namespace |
| 4 | Argo CD creates the database, backend and frontend workloads |
| 5 | Each pod gets its interfaces: `ovn-udn1` on `10.50.0.0/16`, `net1` on `192.168.50.0/24` |

> [!WARNING]
> The `k8s.ovn.org/primary-user-defined-network` label only works if it is on the namespace
> **when the namespace is created**, it cannot be added later. And the primary CUDN must exist
> **before** pods start, otherwise they stay on the default network. The sync waves take care
> of both.

The chart renders the objects below. You don't apply them by hand when using Argo CD, they
are here so you can see exactly what gets created.

### 4.1 Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-frontend
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
---
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-backend
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
    app-tier: webapp
    k8s.ovn.org/primary-user-defined-network: ""
---
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-database
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
    app-tier: webapp
    k8s.ovn.org/primary-user-defined-network: ""
```

### 4.2 Primary CUDN

`webapp-cudn` for the backend and database.

```yaml
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: webapp-cudn
spec:
  namespaceSelector:
    matchLabels:
      app-tier: webapp
  network:
    topology: Layer2
    layer2:
      role: Primary
      subnets:
        - "10.50.0.0/16"
      ipam:
        lifecycle: Persistent
```

### 4.3 Secondary CUDN

`frontend-backend-net` for the frontend and backend.

```yaml
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: frontend-backend-net
spec:
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
          - webapp-frontend
          - webapp-backend
  network:
    topology: Layer2
    layer2:
      role: Secondary
      subnets:
        - "192.168.50.0/24"
```

Pods join it through this annotation (already set on the frontend and backend Deployments):

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: frontend-backend-net
```

### 4.4 NetworkPolicy

Only the backend may reach the database.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: restrict-db-access
  namespace: webapp-database
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app-tier: webapp
          podSelector:
            matchLabels:
              app: backend-api
```

### 4.5 Check the networks after deployment

```bash
oc get clusteruserdefinednetworks
oc get network-attachment-definitions -A | grep -E 'webapp-cudn|frontend-backend-net'
oc get ns webapp-frontend webapp-backend webapp-database --show-labels

```


---

## 5. Deploy the Helm chart with Argo CD

Argo CD pulls the Helm chart from GitLab (`root/prime-app`, path `charts/prime-app`) and deploys it to the cluster.

### 5.1 Register the repository in Argo CD

Save as `prime-app-repo.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-prime-app
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.apps.ocp4.example.com/root/prime-app.git
  username: root
  password: <argocd-read-token>
  insecure: "true"      # lab self-signed certificate
```

```bash
oc apply -f prime-app-repo.yaml
```

### 5.2 Let Argo CD manage the CUDNs

The `argocd.argoproj.io/managed-by` label on each namespace covers the namespaced objects. The cluster-scoped CUDNs need extra RBAC. Save as `prime-app-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prime-app-argocd
rules:
  - apiGroups: ["k8s.ovn.org"]
    resources: ["clusteruserdefinednetworks"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prime-app-argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prime-app-argocd
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
```

```bash
oc apply -f prime-app-rbac.yaml
```

### 5.3 Create the Application

Save as `prime-app-application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prime-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://gitlab.apps.ocp4.example.com/root/prime-app.git
    targetRevision: main
    path: charts/prime-app
    helm:
      releaseName: prime-app
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: webapp-frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
    retry:
      limit: 10
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

```bash
oc apply -f prime-app-application.yaml
```

The chart creates the resources in the order set by each resource's `argocd.argoproj.io/sync-wave` annotation: 1 Namespaces, 2 Networks, 3 Database, 4 Backend, 5 Frontend.

### 5.4 Watch the sync

```bash
oc get application prime-app -n openshift-gitops -w       # wait for Synced / Healthy

# details, or the error if it is stuck
oc get application prime-app -n openshift-gitops -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}{.status.conditions[*].message}{"\n"}'
```

In the Argo CD UI, the `prime-app` tile shows **Healthy** and **Synced**, with path `charts/prime-app` and namespace `webapp-frontend`.

---
<img width="1917" height="1008" alt="image" src="https://github.com/user-attachments/assets/991698a7-f050-4e5f-af25-56dc59059c70" />

<img width="1916" height="1010" alt="image" src="https://github.com/user-attachments/assets/b543fb79-e704-4fe0-8d22-bbe6c404daec" />


---

## 6. Verify

### 6.1 Argo CD

```bash
oc get application prime-app -n openshift-gitops
oc get application prime-app -n openshift-gitops -o jsonpath='{range .status.resources[*]}{.kind}{"\t"}{.namespace}{"\t"}{.name}{"\t"}{.status}{"\t"}{.health.status}{"\n"}{end}' | column -t
```

### 6.2 Pods

```bash
oc get pods -n webapp-database     # webapp-db-0            1/1 Running
oc get pods -n webapp-backend      # backend-api-...        1/1 Running
oc get pods -n webapp-frontend     # frontend-web-...       2/2 Running 

```

### 6.3 End to end

```bash
HOST=$(oc get route prime-series-movies -n webapp-frontend -o jsonpath='{.spec.host}')
echo "http://$HOST"
curl -s "http://$HOST/api/health"; echo
# {"status": "ok", "database": "up", ..., "titles": 16}
```



### 6.4 Every network path

```bash
chmod +x scripts/verify-tiers.sh
./scripts/verify-tiers.sh
```

It checks every row of [Who can reach what](#-who-can-reach-what) and ends with
`N passed, 0 failed`. Two short-lived test pods prove the NetworkPolicy: the same pod is
blocked with `app=something-else` and allowed with `app=backend-api`.

---

## 7. Day-2 operations

###  Change anything through Git

```bash
sed -i 's/^  replicas: 2$/  replicas: 3/' charts/prime-app/values.yaml     # backend replicas
git commit -am "Scale backend to 3" && git push gitlab main

# sync now instead of waiting for the 3-minute poll
oc annotate application prime-app -n openshift-gitops argocd.argoproj.io/refresh=normal --overwrite
oc get pods -n webapp-backend -w
```



###  Watch self-heal undo a manual change

```bash
oc scale deploy/backend-api -n webapp-backend --replicas=5
oc get deploy backend-api -n webapp-backend -w           # goes back to the value in Git
```

###  Watch the sidecar follow the backend

```bash
oc delete pod -n webapp-backend -l app=backend-api --wait=false
oc logs -n webapp-frontend deploy/frontend-web -c upstream-sync -f   # new IPs + "reloaded nginx"
```



---



## 8. Troubleshooting

| Errors Example | Cause | Fix |
|---|---|---|
| `ErrImagePull … lookup <registry>: no such host` | Registry name from another environment | Use images the nodes can reach  |
| `x509: certificate is valid for git.ocp4.example.com, not registry…` | Registry port missing, so port 443 (Git server) was hit | Add the port, e.g. `registry.ocp4.example.com:8443/...` |
| `unauthorized: access to the requested resource is not authorized` | Image not in that registry, or private | Check the path; Quay answers "unauthorized" for missing repos too |
| `This repo requires terms acceptance and is only available on registry.redhat.io` | RHEL image requested from `registry.access.redhat.com` | Use `registry.redhat.io/...` (global pull secret) |
| `CreateContainerError … does not resolve to an image ID` | Broken image entry in CRI-O storage on the node | `oc debug node/<node> -- chroot /host crictl rmi <image>` then delete the pod |
| `Table 'primedb.titles' doesn't exist` | Schema not loaded | The StatefulSet's postStart hook loads it on start; see the command below |
| PVC `data-webapp-db-0` stays `Pending` | No default StorageClass | Set `database.storage.storageClassName` |
| `git push`: `HTTP Basic: Access denied` | Password used instead of a token | Use a personal access token  |
| `git push`: `requires … permissions: [Code: Push]` / `[Code: Download]` | Fine-grained token missing a permission | Add **Code: Push** and **Code: Download** |
| `! [rejected] main -> main (fetch first)` | GitLab project was created with a README | `git pull gitlab main --allow-unrelated-histories --no-rebase -X ours --no-edit`, then push |
| Argo CD UI shows no applications | User not in `cluster-admins` | , then log out and in |
| Application `Unknown`: `failed to list refs: authentication required` | Wrong token or repository Secret | Test with `git ls-remote`  and recreate the Secret |
| First sync: `… is forbidden: User "system:serviceaccount:openshift-gitops:…"` | Namespaces just created, RBAC not granted yet | Wait for the retry; check the `argocd.argoproj.io/managed-by` label |
| Site footer: "Backend not reachable" | No ready backend pods, or sidecar can't list them | `oc logs deploy/frontend-web -c upstream-sync -n webapp-frontend` |
| Health says `"database": "down"` | Backend can't reach MariaDB | `oc logs deploy/backend-api -n webapp-backend --tail=20`; check pod label `app=backend-api` |



---

## 9. Application UI

<img width="1916" height="1015" alt="image" src="https://github.com/user-attachments/assets/63431d5a-af6f-4c8c-aa94-7dcee550def5" />

---
<div align="center">

Built with  OpenShift · 🔗 OVN-Kubernetes user-defined networks · ⛵ Helm · 🐙 Argo CD · 🦊 GitLab

</div>
