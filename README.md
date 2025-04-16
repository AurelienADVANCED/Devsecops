# Projet E5 DevSecOps – ESTIAM Metz  
## Groupe WMD – Rapport de réalisation  
**Nom : Aurelien ROSELLO**  
**Binôme : Aurian BOHN**  
**Date : 16 avril 2025**

## 📚 Sommaire

1. [🎯 Objectif du projet](#-objectif-du-projet)  
2. [🏗️ Architecture cible](#architecture-cible)
3. [📁 Organisation du projet](#-organisation-du-projet)  
4. [🔧 Étapes de réalisation](#-étapes-de-réalisation)  
5. [🐳 Création des images Docker](#-création-des-images-docker)  
6. [🚀 Optimisation des images Docker](#-optimisation-de-limage-docker-django)  
7. [🌐 Ingress Kubernetes](#-ingress-kubernetes)  
8. [📦 Déploiement des applications Kubernetes](#-déploiement-des-applications-kubernetes)  
9. [🚀 Commandes de déploiement Kubernetes](#-commandes-de-déploiement-kubernetes)  
10. [✅ Conclusion](#-conclusion)

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
  /django        /node (NodePort)  /flask (LB)
  Django          Next.js            Flask
   Port 80       Port 9090          Port 8080

```


### 🌐 Résumé des accès

| Application | Port exposé | Type de service | Accès via Ingress             | Accès direct             |
|-------------|-------------|------------------|-------------------------------|---------------------------|
| Django      | 80          | ClusterIP        | `http://projet.local/django`  | `http://<minikube-ip>:80`        |
| Next.js     | 9090        | NodePort         | `http://projet.local/node`    | `http://<minikube-ip>:30080` |
| Flask       | 8080        | LoadBalancer     | `http://projet.local/flask`   | `http://<minikube-ip>:8080` (via tunnel) |

---
## 📁 Organisation du projet

![image](https://github.com/user-attachments/assets/47765e81-565c-485c-b5c5-fa95039b50af)

📦 Chaque application dispose de son propre dossier avec son `Dockerfile`.  
📂 Tous les fichiers Kubernetes (`Deployment`, `Service`, `Ingress`) sont regroupés dans le dossier `k8s/` pour plus de clarté.

---

## 🔧 Étapes de réalisation

### 🔹 1. Initialiser Minikube

```bash
minikube start
minikube addons enable ingress
minikube tunnel
```

## 🐳 Création des images Docker

### 📦 1. Application critique – Django

**Fichier : django-volt/Dockerfile**

```dockerfile
FROM python:3.9

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

COPY requirements.txt .
# install python dependencies
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Set UP
RUN python manage.py collectstatic --no-input
RUN python manage.py makemigrations
RUN python manage.py migrate

#__API_GENERATOR__
#__API_GENERATOR__END

# Start Server
EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

# Commande de build :
```
docker build -t localhost/django-app:latest .
```

# Charger l'image dans Minikube :
```
minikube image load localhost/django-app:latest
```

![image](https://github.com/user-attachments/assets/c264f25f-557f-4e4c-ba0e-772e79b314c1)

## 🚀 Optimisation de l'image Docker Django

Afin de produire une image **plus légère, plus rapide à déployer et plus sécurisée**, l’image Docker de l’application Django a été optimisée à l’aide d’un **build multi-étapes**.

### 🧱 Étape 1 : Build intermédiaire avec compilation

```dockerfile
FROM python:3.9-slim AS builder

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

WORKDIR /app
COPY requirements.txt .

# Installation des outils de compilation + dépendances
RUN apt-get update \
  && apt-get install -y build-essential gcc \
  && pip install --upgrade pip \
  && pip install --user --no-warn-script-location --no-cache-dir -r requirements.txt \
  && apt-get remove -y build-essential gcc \
  && apt-get autoremove -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*
```
### 📦 Étape 2 : Image finale minimaliste

```dockerfile
FROM python:3.9-slim

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

WORKDIR /app

# Récupération des paquets Python installés depuis l'étape précédente
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copie du code source Django
COPY . .

# Préparation des fichiers statiques + migrations
RUN python manage.py collectstatic --no-input && \
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
![image](https://github.com/user-attachments/assets/57576e4b-8513-43d7-9a08-117c79ead657)

# 📦 2. Application secondaire – Next.js
## Fichier : hello-world-next-js/Dockerfile
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 8080
CMD ["npm", "start"]
```
## Commande de build :
```
docker build -t localhost/nextjs-app:1.0 .
```

## Charger l’image dans Minikube :
```
docker build -t localhost/nextjs-app:1.0 .
```

🔐 .dockerignore
```dockerfile
node_modules/
.next/
.env
*.log
.git/
.gitignore
Dockerfile
docker-compose.yml
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

# Port exposé : 8080 (selon les besoins du client)
EXPOSE 8080

# Lancement de l’app avec Gunicorn en mode production
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "run:app"]
```

# Commande de build :
```
docker build -t localhost/flask-app:latest .
```

# Charger l'image dans Minikube :
```
minikube image load localhost/flask-app:latest
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

![image](https://github.com/user-attachments/assets/f13d0ec4-9251-4f75-8c7d-147ca1d14817)

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
          - path: /django
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
                name: next-js-service
                port:
                  number: 9090
```

![image](https://github.com/user-attachments/assets/61c8cab3-5f11-4651-b1ee-567f81136825)

# 📦 Déploiement des applications Kubernetes

## ⚙️ Déploiement de l'application Node.js (port 8080)

**Fichier : `k8s/node-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node
  template:
    metadata:
      labels:
        app: node
    spec:
      containers:
        - name: node
          image: node-app:1.0
          ports:
            - containerPort: 8080
```

Fichier : k8s/node-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-service
spec:
  type: NodePort
  selector:
    app: node
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

## ⚙️ Déploiement de l'application statique NGINX (port 9090)
Fichier : k8s/nginx-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx-app:1.0
          ports:
            - containerPort: 9090
```
Fichier : k8s/nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 9090
      targetPort: 9090
```

## 🚀 Commandes de déploiement Kubernetes

Voici les commandes à exécuter pour déployer tous les composants dans le cluster :

```bash
# Déployer l'Ingress
k apply -f k8s/ingress.yml

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

## ✅ Conclusion

Ce projet nous a permis de mettre en place une maquette fonctionnelle d’un cluster Kubernetes local, capable d’héberger et de faire cohabiter trois applications web distinctes : Django (critique), Next.js, et Flask.

Grâce à l’utilisation de fichiers `Dockerfile` adaptés, d’images optimisées et de manifests Kubernetes séparés pour chaque composant, nous avons pu respecter les exigences de modularité, de portabilité et de scalabilité imposées par le client.

L’usage d’un Ingress unique a facilité l’accès aux services via une seule URL (`projet.local`), tout en respectant les règles de routage propres à chaque application. Le tout est documenté, versionné, et prêt à être migré dans un environnement de production cloud ou intégré à une pipeline CI/CD.

🎯 **Compétences mobilisées** :
- Docker & optimisation des images
- Architecture Kubernetes (Minikube, Services, Ingress, etc.)
- Gestion multi-applications et exposition centralisée
- Automatisation du déploiement

Le projet est prêt pour une démonstration, un audit technique, ou une future industrialisation.
