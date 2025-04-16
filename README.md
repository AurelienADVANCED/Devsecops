# Projet E5 DevSecOps – ESTIAM Metz  
## Groupe WMD – Rapport de réalisation  
**Nom : Aurelien ROSELLO**  
**Date : 16 avril 2025**

---

## 🎯 Objectif du projet

L’objectif est de créer une maquette de cluster Kubernetes capable d’héberger trois applications, dont une critique développée en Django, chacune accessible via un port différent (80, 8080, 9090).  
Chaque composant doit être défini dans un manifeste séparé, et toutes les images Docker doivent être créées et hébergées dans un dépôt Git.

---

## 🏗️ Architecture cible
```
       Internet
           ↓
    [Ingress NGINX]
    ┌────┬─────┬
    ↓    ↓     ↓
 Django Node  Nginx
  (80)   (8080) (9090)
```
---
```
## 📁 Organisation du projet

projet-devsecops/
├── django-app/                  # Application critique (port 80)
│   ├── app/                     # Code Django
│   ├── requirements.txt         # Dépendances Python
│   ├── Dockerfile               # Image Docker de l'app
│   └── .env                     # Variables d'environnement (non versionnées)
│
├── node-app/                    # Application secondaire (port 8080)
│   ├── server.js                # Serveur Node.js simple
│   └── Dockerfile               # Image Docker
│
├── nginx-app/                   # Troisième application (port 9090)
│   ├── index.html               # Page HTML statique
│   └── Dockerfile               # Image Docker avec nginx
│
├── k8s/                         # Manifeste Kubernetes
│   ├── django-deployment.yaml
│   ├── django-service.yaml
│   ├── node-deployment.yaml
│   ├── node-service.yaml
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   └── ingress.yaml             # Exposition via Ingress
│
├── README.md                    # Instructions du projet
└── .gitignore                   # Fichiers à exclure du repo
```
---

## 🔧 Étapes de réalisation

### 🔹 1. Initialiser Minikube

```bash
minikube start
minikube addons enable ingress
minikube tunnel
```

## 🐳 Création des images Docker

### 📦 1. Application critique – Django (port 80)

**Fichier : `django-app/Dockerfile`**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:80", "app.wsgi"]
```

# Commande de build :
docker build -t django-app:1.0 ./django-app

# 📦 2. Application Node.js (port 8080)
## Fichier : node-app/Dockerfile
```dockerfile
FROM node:18-slim
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["node", "server.js"]
```
## Commande de build :
```
docker build -t node-app:1.0 ./node-app
```

# 📦 3. Application statique NGINX (port 9090)
## Fichier : nginx-app/Dockerfile
```dockerfile
FROM nginx:alpine
COPY ./index.html /usr/share/nginx/html/index.html
```

# Commande de build :
```
docker build -t nginx-app:1.0 ./nginx-app
```

## 🌐 Ingress Kubernetes

Le fichier suivant permet de définir un Ingress unique exposant les trois applications sur un seul domaine (`projet.local`) avec des chemins distincts pour chaque service.

**Fichier : `k8s/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: projet-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: projet.local
    http:
      paths:
      - path: /django
        pathType: Prefix
        backend:
          service:
            name: django-service
            port:
              number: 80
      - path: /node
        pathType: Prefix
        backend:
          service:
            name: node-service
            port:
              number: 8080
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 9090
```
