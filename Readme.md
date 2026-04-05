<!-- ===============================
      Badges Section
=============================== -->
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=node.js&logoColor=white)
![Express.js](https://img.shields.io/badge/Express.js-000000?style=for-the-badge&logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

---

<!-- ===============================
      Description
=============================== -->
## Description
This is a full-stack project using **MongoDB, Express.js, Node.js**, deployed with **Kubernetes** and **Docker**. The app demonstrates containerized deployment, database integration, and scalable microservice architecture.

---

<!-- ===============================
      Recommended GitHub Topics
=============================== -->
**Topics to add in GitHub About section:**  

# 🚀 Kubernetes: MongoDB + Mongo Express সেটআপ গাইড

**তারিখ:** ২০২৫  
**প্রজেক্ট:** `practise-kubernetes`  
**উদ্দেশ্য:** Kubernetes-এ MongoDB ডেটাবেজ এবং Mongo Express UI চালানো

---

## 📁 প্রজেক্টের ফাইল স্ট্রাকচার

```
practise-kubernetes/
├── mongo-secret.yaml       # ১ম — Secret (username/password)
├── mongo.yaml              # ২য় — MongoDB Deployment + Service
├── mongo-configmap.yaml    # ৩য় — ConfigMap (database URL)
└── mongo-express.yaml      # ৪র্থ — Mongo Express Deployment + Service
```

> [!important] ফাইল তৈরির ক্রম অনুসরণ করতে হবে
> Kubernetes-এ যে রিসোর্স আগে দরকার, সেটা আগে apply করতে হয়। Secret আগে না করলে Deployment চালু হবে না।

---

## 🧠 মূল ধারণা (Core Concepts)

| রিসোর্স | কাজ |
|---|---|
| **Secret** | সংবেদনশীল তথ্য (username, password) base64 এনকোড করে রাখে |
| **ConfigMap** | সাধারণ কনফিগ ডেটা রাখে (যেমন: database URL) |
| **Deployment** | কোন Pod কীভাবে চলবে সেটা নির্ধারণ করে |
| **Service (ClusterIP)** | ক্লাস্টারের ভেতরে Pod-এ অ্যাক্সেস দেয় |
| **Service (LoadBalancer)** | বাইরে থেকে অ্যাক্সেস দেয় (Minikube-এ tunnel লাগে) |

---

## ধাপ ১ — Secret তৈরি করা (`mongo-secret.yaml`)

### ফাইলের কাজ
MongoDB-এর root username এবং password গোপনভাবে সংরক্ষণ করা।

### ফাইল কন্টেন্ট
```yaml
apiVersion: v1
kind: Secret
metadata: 
  name: mongodb-secret
type: Opaque
data: 
  mongo-root-username: cm9vdA==   # base64("root")
  mongo-root-password: cGFzcw==  # base64("pass")
```

> [!tip] Base64 এনকোড কীভাবে করবে?
> টার্মিনালে: `echo -n 'root' | base64` → `cm9vdA==`  
> টার্মিনালে: `echo -n 'pass' | base64` → `cGFzcw==`

### কমান্ড
```bash
kubectl apply -f mongo-secret.yaml
```

### আউটপুট
```
secret/mongodb-secret unchanged
```

---

## ধাপ ২ — MongoDB Deployment + Service (`mongo.yaml`)

### ফাইলের কাজ
- **Deployment:** MongoDB-এর Pod চালু করা  
- **Service (ClusterIP):** ক্লাস্টারের ভেতরে MongoDB-কে `mongodb-service` নামে অ্যাক্সেসযোগ্য করা

### ফাইল কন্টেন্ট
```yaml
# --- Deployment ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret 
              key: mongo-root-password 

---
# --- Service ---
apiVersion: v1
kind: Service 
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb 
  ports:
  - protocol: TCP 
    port: 27017 
    targetPort: 27017
```

> [!note] একই ফাইলে দুটো রিসোর্স
> `---` দিয়ে আলাদা করলে একটা ফাইলেই Deployment এবং Service দুটো রাখা যায়। এটা একটা সুবিধাজনক প্র্যাকটিস।

### কমান্ড
```bash
kubectl apply -f mongo.yaml
```

### আউটপুট
```
deployment.apps/mongodb-deployment created
service/mongodb-service created
```

---

## ধাপ ৩ — ConfigMap তৈরি করা (`mongo-configmap.yaml`)

### ফাইলের কাজ
Mongo Express জানবে কোন address-এ MongoDB আছে — এই তথ্যটা ConfigMap-এ রাখা হয়েছে।

### ফাইল কন্টেন্ট
```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: mongodb-configmap
data: 
  database-url: mongodb-service
```

> [!info] `database-url` এর মান কেন `mongodb-service`?
> Kubernetes-এ Pod গুলো Service-এর **নাম** দিয়ে একে অপরের সাথে কথা বলে। `mongodb-service` হলো ধাপ ২-এ তৈরি করা Service-এর নাম।

### কমান্ড
```bash
kubectl apply -f mongo-configmap.yaml
```

### আউটপুট
```
configmap/mongodb-configmap created
```

---

## ধাপ ৪ — Mongo Express Deployment + Service (`mongo-express.yaml`)

### ফাইলের কাজ
- **Deployment:** Mongo Express UI-এর Pod চালু করা  
- **Service (LoadBalancer):** Browser থেকে Mongo Express-এ ঢোকার রাস্তা তৈরি করা

### ফাইল কন্টেন্ট
```yaml
# --- Deployment ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express 
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports: 
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap  
              key: database-url
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret 
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret  
              key: mongo-root-password

---
# --- Service ---
apiVersion: v1
kind: Service 
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer 
  ports:
  - protocol: TCP 
    port: 8081  
    targetPort: 8081
    nodePort: 30000
```

### কমান্ড
```bash
kubectl apply -f mongo-express.yaml
```

### আউটপুট
```
deployment.apps/mongodb-express-deployment created
```

---

## ধাপ ৫ — Pod এবং Service যাচাই করা

### Pod চেক করা
```bash
kubectl get pods
```

**আউটপুট (সফল):**
```
NAME                                  READY   STATUS    RESTARTS   AGE
mongo-express-c8fd4c668-pk6ll         1/1     Running   0          19s
mongodb-deployment-7f7d7cc59b-79kfd   1/1     Running   0          38m
```

### Service চেক করা
```bash
kubectl get services
```

**আউটপুট:**
```
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes              ClusterIP      10.96.0.1        <none>        443/TCP          34h
mongo-express-service   LoadBalancer   10.96.126.126    <pending>     8081:30000/TCP   64s
mongodb-service         ClusterIP      10.100.111.103   <none>        27017/TCP        46m
```

> [!warning] `EXTERNAL-IP` কেন `<pending>`?
> Minikube local environment-এ LoadBalancer Service-এর External IP সরাসরি আসে না। এজন্য `minikube service` কমান্ড ব্যবহার করতে হয়।

---

## ধাপ ৬ — Browser-এ Mongo Express খোলা

```bash
minikube service mongo-express-service
```

**আউটপুট:**
```
┌───────────┬───────────────────────┬─────────────┬───────────────────────────┐
│ NAMESPACE │         NAME          │ TARGET PORT │            URL            │
├───────────┼───────────────────────┼─────────────┼───────────────────────────┤
│ default   │ mongo-express-service │ 8081        │ http://192.168.49.2:30000 │
└───────────┴───────────────────────┴─────────────┴───────────────────────────┘
🔗  Starting tunnel for service mongo-express-service.
```

Browser-এ যাও: `http://192.168.49.2:30000`

**Login তথ্য:**
```
Username: admin
Password: pass
```

---

## ⚠️ যে ভুল হয়েছিল এবং সমাধান

### সমস্যা: `CreateContainerConfigError`

**কারণ:** `mongo-express.yaml`-এ Secret-এর নাম ভুল ছিল।

| ফাইল | ভুল নাম | সঠিক নাম |
|---|---|---|
| `mongo-express.yaml` | `mongo-secret` | `mongodb-secret` |
| `mongo-secret.yaml` | — | `mongodb-secret` ✅ |

**শিক্ষা:** Secret, ConfigMap-এর `name` এবং Deployment-এর `secretKeyRef.name` / `configMapKeyRef.name` সবসময় **হুবহু একই** হতে হবে।

---

## 🔄 পুরো ফ্লো একনজরে

```
mongo-secret.yaml ──────────────────────────────────┐
     ↓ (username/password সংরক্ষণ)                  │
mongo.yaml ─────────────────────────────────────────┤
     ↓ (MongoDB Pod চালু + ClusterIP Service)        │
mongo-configmap.yaml ───────────────────────────────┤
     ↓ (database URL সংরক্ষণ)                       │
mongo-express.yaml ─────────────────────────────────┘
     ↓ (Mongo Express Pod চালু + LoadBalancer Service)
minikube service mongo-express-service
     ↓
Browser → http://192.168.49.2:30000
```

---

## 📋 দরকারি কমান্ড রেফারেন্স

```bash
# সব Pod দেখা
kubectl get pods

# সব Service দেখা
kubectl get services

# সব Secret দেখা
kubectl get secrets

# সব ConfigMap দেখা
kubectl get configmaps

# Pod-এর log দেখা
kubectl logs <pod-name>

# Pod-এর বিস্তারিত দেখা (ভুল ধরতে কাজে লাগে)
kubectl describe pod <pod-name>

# রিসোর্স মুছে ফেলা
kubectl delete -f <filename>.yaml

# Minikube দিয়ে Browser-এ Service খোলা
minikube service <service-name>
```

---

## 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                      │
│  ┌──────────────────┐    ┌───────────────────────┐  │
│  │  Mongo Express   │    │       MongoDB         │  │
│  │     Pod          │───▶│        Pod            │  │
│  │  (port: 8081)    │    │    (port: 27017)      │  │
│  └────────┬─────────┘    └───────────┬───────────┘  │
│           │                          │               │
│  ┌────────▼─────────┐    ┌───────────▼───────────┐  │
│  │ mongo-express-   │    │   mongodb-service     │  │
│  │    service       │    │   (ClusterIP)         │  │
│  │ (LoadBalancer)   │    │   port: 27017         │  │
│  │ port: 8081:30000 │    └───────────────────────┘  │
│  └────────┬─────────┘                               │
└───────────┼─────────────────────────────────────────┘
            │
            ▼
     Browser (তোমার PC)
  http://192.168.49.2:30000
```

---

