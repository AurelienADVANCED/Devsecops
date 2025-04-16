# Projet E5 DevSecOps – ESTIAM Metz  
## Groupe WMD – Rapport de réalisation  
**Groupe : Aurelien ROSELLO & Aurian BOHN**

**Date : 16 avril 2025**

## 📚 Sommaire

1. [🎯 Objectif du projet](#-objectif-du-projet)  
2. [🏗️ Architecture cible](#architecture-cible)
3. [📁 Organisation du projet](#-organisation-du-projet)  
4. [🔧 Étapes de réalisation](#-étapes-de-réalisation)  
5. [🐳 Création des images Docker](#-création-des-images-docker)  
6. [🚀 Optimisation des images Docker](#-optimisation-de-limage-docker-django)  
7. [📦 Déploiement des applications Kubernetes](#-déploiement-des-applications-kubernetes)  
8. [🌐 Ingress Kubernetes](#-ingress-kubernetes)  
9. [🚀 Commandes de déploiement Kubernetes](#-commandes-de-déploiement-kubernetes)
10. [ℹ️ Concepts DevSecOps & Kubernetes](#️-concepts-devsecops--kubernetes)  
11. [✅ Conclusion](#-conclusion)

---

## 🎯 Objectif du projet

L'objectif de ce projet est de concevoir une maquette fonctionnelle d’un cluster Kubernetes local capable d’héberger trois applications web distinctes, dont une application critique développée en Django.  
Chaque application doit être exposée sur un port spécifique (80, 8080 et 9090) et déployée à l’aide de fichiers manifeste Kubernetes indépendants.  
Toutes les images Docker doivent être construites manuellement, optimisées, et intégrées dans un dépôt Git afin d’assurer la traçabilité et la portabilité de l’environnement.

---

<a name="architecture-cible"></a>
## 🏗️ Architecture cible

L’architecture repose sur un cluster Kubernetes local (via Minikube), hébergeant trois applications clonées et adaptées à partir de dépôts publics :

- ✅ **Django Volt** – [django-volt-1744777679](https://github.com/app-generator/django-volt-1744777679)  
- ✅ **Next.js** – [hello-world-next-js](https://github.com/app-generator/hello-world-next-js)  
- ✅ **Flask** – [flask-soft-1744678708](https://github.com/app-generator/flask-soft-1744678708)  

### 🔀 Schéma logique

```
           Navigateur / Client
                    │
                    ▼
            [ Ingress NGINX ]
                    │
   ┌────────────────┼────────────────┐
   │                │                │
   ▼                ▼                ▼
  /django        /front            /flask
  Django          Next.js           Flask
   Port 80       Port 9090         Port 8080

```


## 🔄 Accès aux applications via port-forward (sans Ingress)

Chaque application est exposée en local sur les ports spécifiés dans la consigne :

| Application | Port exposé | Type de service | Accès via Ingress             |
|-------------|-------------|------------------|-------------------------------|
| Django      | 80          | ClusterIP        | `http://localhost/django`  |
| Next.js     | 9090        | ClusterIP         | `http://localhost/front`    |
| Flask       | 8080        | ClusterIP     | `http://localhost/flask`   |

💡 Ces redirections permettent d'accéder directement à :

- `http://localhost:80` → Django  
- `http://localhost:8080` → Flask  
- `http://localhost:9090` → Next.js  

> 🧠 Faire ces commandes dans **3 terminaux séparés**.
---
## 📁 Organisation du projet

![archi](https://github.com/user-attachments/assets/bfe1808c-635e-4e2f-ab4c-ad056e292bce)

![image](https://github.com/user-attachments/assets/87e260df-18c0-4d26-a4f1-fa5b158c9140)

📦 Chaque application dispose de son propre dossier avec son `Dockerfile`.  
📂 Tous les fichiers Kubernetes (`Deployment`, `Service`, `Ingress`) sont regroupés dans le dossier `k8s/` pour plus de clarté.

---

## 🔧 Étapes de réalisation

### 🔹 1. Initialiser Minikube

```bash
minikube start
minikube addons enable ingress
```

## 🐳 Création des images Docker

### 📦 1. Application critique – Django

**Fichier : django-volt/Dockerfile**

```dockerfile
FROM python:3.11-alpine

# Env vars
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Dépendances système
RUN apk update && apk add --no-cache \
    gcc \
    python3-dev \
    musl-dev \
    libffi-dev \
    jpeg-dev \
    zlib-dev \
    postgresql-dev \
    build-base \
    linux-headers \
    libxml2-dev \
    libxslt-dev \
    && pip install --upgrade pip

WORKDIR /app

# Copier uniquement requirements.txt pour tirer parti du cache Docker
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

# Copier le code de l'app
COPY . .

# Collecte des fichiers statiques & migrations
RUN python manage.py collectstatic --noinput && \
    python manage.py makemigrations && \
    python manage.py migrate

EXPOSE 5005

CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

🔐 .dockerignore
```dockerfile
# Répertoires de dev
__pycache__/
*.py[cod]
*.sqlite3
*.log
*.env
*.db
*.bak
*.swp

# Répertoires spécifiques
.env/
venv/
env/
.idea/
.vscode/
.mypy_cache/
.coverage
node_modules/

# Git
.git/
.gitignore

# Fichiers générés par collectstatic
staticfiles/
static/

# Docker
Dockerfile
.dockerignore
```

# Commande de build :
```
docker build -t localhost/django-app:latest .
```

# 📦 2. Application secondaire – Next.js
## Fichier : hello-world-next-js/Dockerfile
```dockerfile
# Étape 1 : Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json ./
RUN npm install

COPY . .
RUN NODE_OPTIONS=--openssl-legacy-provider npm run build

# Étape 2 : Image de production
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev

COPY --from=builder /app/.next .next
COPY --from=builder /app/pages pages
COPY --from=builder /app/node_modules node_modules
COPY --from=builder /app/package.json ./

EXPOSE 9090

CMD ["npx", "next", "start", "-p", "9090"]
```

🔐 .dockerignore
```dockerfile
node_modules
npm-debug.log
.next
Dockerfile
.dockerignore
.git
.gitignore
.vscode
```

## Commande de build :
```
docker build -t localhost/next-js-app:latest .
```

# 3. Application Flask
## Fichier : flask-soft/Dockerfile
```dockerfile
FROM python:3.11-alpine

# Eviter l'écriture de fichiers pyc + affichage live dans le terminal
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Installation des dépendances système
RUN apk update && apk add --no-cache \
    build-base \
    python3-dev \
    libffi-dev \
    musl-dev \
    gcc \
    jpeg-dev \
    zlib-dev \
    postgresql-dev \
    mariadb-dev \
    linux-headers

# Dossier de travail
WORKDIR /app

# Copier requirements en premier pour profiter du cache
COPY requirements.txt .

# Installer les dépendances
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copier tout le reste du code
COPY . .

# Port exposé : 8080 
EXPOSE 8080

# Lancement de l’app avec Gunicorn en mode production
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "run:app"]
```

🔐 .dockerignore
```dockerfile
 docker
# Python
__pycache__/
*.py[cod]
*.log
*.sqlite3
*.env
*.db
*.swp
*.bak

# Environnement virtuel
venv/
env/

# IDE & outils
.vscode/
.idea/
.mypy_cache/

# Node.js (si jamais intégré pour frontend)
node_modules/

# Git
.git/
.gitignore

# Media & statiques générés
media/
static/

# Docker
Dockerfile
.dockerignore
```

# Commande de build :
```
docker build -t localhost/flask-app:latest .
```


## 🌐 Ingress Kubernetes

Le fichier suivant permet de définir un Ingress unique exposant les trois applications sur un seul domaine avec des chemins distincts pour chaque service.

**Fichier : `k8s/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-service
                port:
                  number: 80
          - path: /flask
            pathType: Prefix
            backend:
              service:
                name: flask-service
                port:
                  number: 8080
          - path: /front
            pathType: Prefix
            backend:
              service:
                name: next-service
                port:
                  number: 9090
```

🔍 Commande minikube service list
Cette commande permet d'afficher tous les services Kubernetes déployés dans le cluster Minikube, ainsi que leurs informations d'exposition (ClusterIP, NodePort ou LoadBalancer).
![image](https://github.com/user-attachments/assets/5407b0df-49aa-402a-b431-a987de59fe07)

🔒 Accès restreint par Minikube
Nous avons effectué un port-forward sur le port 80 (utilisé initialement par l’Ingress), car Minikube ne permet pas d’accéder directement aux adresses IP internes depuis l’extérieur de la VM. Ce port-forward permet ainsi un accès local à l'application via localhost.
![image](https://github.com/user-attachments/assets/c2db467b-7c3d-42dc-bff3-ceecb4224d60)

# 📦 Déploiement des applications Kubernetes

## ⚙️ Déploiement de l'application Django (port 80)
Fichier : k8s/django-deployment.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
        - name: django
          image: localhost/django-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 5005
```

🧱 Détail du fichier k8s/django-deployment.yml
```
kind: Deployment
```
🎯 Définit le type de ressource Kubernetes que tu crées ici.
Deployment sert à déployer, maintenir et mettre à jour des groupes de Pods.


```
spec:
  replicas: 1
```
🔁 Le champ replicas indique le nombre de pods à exécuter.
Ici, 1 pod sera déployé pour l'application Django.

```
  selector:
    matchLabels:
      app: django-app
```
🔎 Ce sélecteur permet au déploiement de trouver les pods à gérer.
Il cible les pods ayant le label app: django-app.


```
    spec:
      containers:
        - name: django
          image: localhost/django-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 5005
```

🔍 Détail de containers

| Champ             | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `name: django`    | Nom interne du container                                                    |
| `image`           | Image Docker à utiliser. Ici : `localhost/django-app:latest`                |
| `imagePullPolicy` | `Never` indique à Kubernetes de ne pas essayer de tirer l'image (locale)    |
| `ports`           | Liste des ports exposés par le container (ici, le port `5005`)              |


Fichier : k8s/django-service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  selector:
    app: django-app
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 5005
```

| Champ        | Description                                                                 |
|--------------|-----------------------------------------------------------------------------|
| `apiVersion` | Version de l'API Kubernetes utilisée. Ici : `v1`                            |
| `kind`       | Type de ressource définie : `Service` signifie qu’on expose une application |
| `metadata`   | Métadonnées (comme `name`) pour identifier le service                       |
| `name`       | Nom du service Kubernetes : `django-service`                                |
| `spec`       | Spécifie la configuration du service (ports, type, selector, etc.)          |
| `selector`   | Associe le service au pod ayant le label `app: django-app`                  |
| `ports`      | Liste des ports pour accéder à l'application                                |
| `port`       | Port exposé par le service (port accessible depuis le cluster)              |
| `targetPort` | Port du container cible (ici `5005`)                                        |
| `type`       | `ClusterIP` signifie que le service est accessible uniquement dans le cluster |

## ⚙️ Déploiement de l'application Next.js (port 9090)
Fichier : k8s/next-js-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: next-app
  template:
    metadata:
      labels:
        app: next-app
    spec:
      containers:
        - name: nextjs
          image: localhost/next-js-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 9090
```
Fichier : k8s/next-js-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: next-service
spec:
  selector:
    app: next-app
  type: LoadBalancer
  ports:
    - port: 9090
      targetPort: 9090
```

## ⚙️ Déploiement de l'application Flask (port 8080)

Fichier : k8s/flask-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: localhost/flask-app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
```

Fichier : k8s/flask-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: 8080
```

## 🚀 Commandes de déploiement Kubernetes

Voici les commandes à exécuter pour déployer tous les composants dans le cluster :

```bash
# Déployer l'application Django
k apply -f k8s/django-deployment.yml
k apply -f k8s/django-service.yml

# Déployer l'application Next.js
k apply -f k8s/next-js-deployment.yml
k apply -f k8s/next-js-service.yml

# Déployer l'application Flask
k apply -f k8s/flask-deployment.yml
k apply -f k8s/flask-service.yml
```

## 🖼️ Captures d'écran du projet

### 🧩 Application 1 : Django

![image](https://github.com/user-attachments/assets/09f903af-7e0d-405e-895b-aa06b5c6c6c7)
![image](https://github.com/user-attachments/assets/c264f25f-557f-4e4c-ba0e-772e79b314c1)

### 🧩 Application 2 : Flask

![image](https://github.com/user-attachments/assets/d600b30e-71ed-454f-a298-79d11d75a8e2)
![image](https://github.com/user-attachments/assets/f248fd82-a62c-4f6d-a775-cdb279b94887)

### 🧩 Application 3 : Next.js

![image](https://github.com/user-attachments/assets/87be0d10-777e-432c-91fd-b6a593aed750)

![image](https://github.com/user-attachments/assets/154ab717-26ab-4853-a6e5-24943d8e0a0c)

## ℹ️ Concepts DevSecOps & Kubernetes

### 🐳 Docker – Conteneurisation

Docker est un outil permettant de **créer, déployer et exécuter des applications dans des conteneurs**. Chaque conteneur regroupe l’application, ses dépendances, et sa configuration. Cela garantit que l’application se comporte de la même manière sur tous les environnements (développement, test, production).

Dans notre projet :
- Chaque application dispose de son propre `Dockerfile`.

### ☸️ Kubernetes – Orchestration de conteneurs

Kubernetes est une plateforme open source qui automatise :
- Le **déploiement** de conteneurs
- Leur **mise à l’échelle**
- La **récupération automatique** en cas de crash
- Le **réseau entre les services**

#### 🔧 Principaux objets Kubernetes utilisés :

| Objet | Description |
|-------|-------------|
| `Pod` | Plus petite unité déployable, contient un ou plusieurs conteneurs. |
| `Deployment` | Gère le cycle de vie des Pods (ex. nombre de réplicas, mises à jour). |
| `Service` | Expose les Pods et permet la communication avec d'autres composants. |
| `Port-forward` | Permet d'accéder aux Services en local depuis un terminal. |

#### 🎛️ Types de Services

- `ClusterIP` : service interne au cluster (par défaut)
- `NodePort` : accessible via l’IP du nœud et un port spécifique (externe)
- `LoadBalancer` : crée un load balancer externe (fonctionne avec Minikube en tunnel ou dans le cloud)

## ✅ Conclusion

Ce projet nous a permis de mettre en place une maquette fonctionnelle d’un cluster Kubernetes local, capable d’héberger et de faire cohabiter trois applications web distinctes : Django (critique), Next.js, et Flask.

Grâce à l’utilisation de fichiers `Dockerfile` adaptés, d’images optimisées et de manifests Kubernetes séparés pour chaque composant, nous avons pu respecter les exigences de modularité, de portabilité et de scalabilité imposées par le client.

🎯 **Compétences mobilisées** :
- Docker & optimisation des images
- Architecture Kubernetes (Minikube, Services, etc.)
- Gestion multi-applications et exposition centralisée
- Automatisation du déploiement

Le projet est prêt pour une démonstration, un audit technique, ou une future industrialisation.
