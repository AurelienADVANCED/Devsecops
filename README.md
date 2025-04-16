# Projet E5 DevSecOps – ESTIAM Metz  
## Groupe WMD – Rapport de réalisation  
**Nom : Aurelien ROSELLO**  
**Date : 15 avril 2025**

---

## 🎯 Objectif du projet

L’objectif est de créer une maquette de cluster Kubernetes capable d’héberger trois applications, dont une critique développée en Django, chacune accessible via un port différent (80, 8080, 9090).  
Chaque composant doit être défini dans un manifeste séparé, et toutes les images Docker doivent être créées et hébergées dans un dépôt Git.

---

## 🏗️ Architecture cible

       Internet
           ↓
    [Ingress NGINX]
    ┌────┬─────┬
    ↓    ↓     ↓
 Django Node  Nginx
  (80)   (8080) (9090)

---

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

---

## 🔧 Étapes de réalisation

### 🔹 1. Initialiser Minikube

```bash
minikube start
minikube addons enable ingress
minikube tunnel
```
