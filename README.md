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
  /django        /front (NodePort)  /flask (LB)
  Django          Next.js            Flask
   Port 80       Port 9090          Port 8080

```


### 🌐 Résumé des accès

| Application | Port exposé | Type de service | Accès via Ingress             |
|-------------|-------------|------------------|-------------------------------|
| Django      | 80          | ClusterIP        | `http://projet.local/django`  |
| Next.js     | 9090        | ClusterIP         | `http://projet.local/front`    |
| Flask       | 8080        | ClusterIP     | `http://projet.local/flask`   |
---
## 📁 Organisation du projet

![image](https://github.com/user-attachments/assets/88a9dbde-33ea-471f-a25e-3fc464af7108)

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

# Port exposé : 8080 (selon les besoins du client)
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


## ⚙️ Déploiement de l'application statique NGINX (port 9090)
Fichier : k8s/nginx-deployment.yaml
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
Fichier : k8s/nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: next-service
spec:
  selector:
    app: next-app
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
  type: ClusterIP
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


![image](https://github.com/user-attachments/assets/f13d0ec4-9251-4f75-8c7d-147ca1d14817)
![image](https://github.com/user-attachments/assets/57576e4b-8513-43d7-9a08-117c79ead657)
![image](https://github.com/user-attachments/assets/c264f25f-557f-4e4c-ba0e-772e79b314c1)
![image](https://github.com/user-attachments/assets/1e16ef1b-b204-468a-ba55-c60bf3f36e4d)

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
