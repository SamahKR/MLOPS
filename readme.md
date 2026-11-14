# 🚀 Real-Time Fraud Detection System (MLOps)

Bienvenue dans ce projet de détection de fraude en temps réel. Ce système complet utilise une architecture MLOps moderne pour ingérer des transactions financières, les enrichir avec un Feature Store, et prédire instantanément si elles sont frauduleuses ou légitimes.

Ce projet a été déployé et testé avec succès sur **Docker Desktop (Kubernetes)**.

## 🏗️ Architecture du Système

Le flux de données fonctionne comme suit :
1. **Simulateur (API FastAPI)** : Génère un flux continu de transactions (avec 20% de fraudes simulées).
2. **Kafka** : Reçoit et transporte les transactions (Message Broker).
3. **Consumer** : Lit les transactions depuis Kafka.
4. **Feast & Redis** : Fournit l'historique de dépenses du client en temps réel (Feature Store).
5. **MLflow** : Fournit le modèle de Machine Learning entraîné (RandomForest).
6. **Décision** : Le Consumer croise les données et affiche `✅ Legit` ou `🚨 FRAUD DETECTED` dans les logs.

---

## 🛠️ Prérequis

Avant de commencer, assurez-vous d'avoir :
*   **Git** (pour cloner le projet)
*   **Docker Desktop** (avec l'option **Kubernetes activée** dans les paramètres)
*   **Python 3.10+** (pour les scripts locaux Feast et ML)
*   **kubectl** (installé et configuré)

---

## 🚀 Guide d'Installation Étape par Étape

### 1. Cloner le projet
```bash
git clone https://github.com/SamahKR/MLOPS.git
cd MLOPS 

### 2.Construire les images Docker

docker build -t real-time-fraud-detection-api:latest -f dockerfiles/Dockerfile.api .
docker build -t real-time-fraud-detection-producer:latest -f dockerfiles/Dockerfile.producer .
docker build -t real-time-fraud-detection-consumer:latest -f dockerfiles/Dockerfile.consumer .

(Note : Docker Desktop partage automatiquement ces images avec Kubernetes, pas besoin de minikube image load).

### 3. Déployer sur Kubernetes

kubectl apply -f k8s-deployment.yaml

Attendez que tous les pods soient en statut Running :
kubectl get pods

### 4. Initialiser Feast (Feature Store)
Générez les données historiques et synchronisez-les vers Redis :
**1. Générer les données Parquet**
python generate_data.py

**2. Enregistrer les features**
cd feature_repo
feast apply
cd ..

**3. Exposer Redis et matérialiser**
kubectl port-forward svc/redis 6379:6379
(Dans un nouveau terminal) :
# Windows PowerShell
$env:REDIS_HOST="localhost"; feast materialize 2026-01-01T00:00:00 2026-12-31T00:00:00
### 5. Entraîner le modèle avec MLflow
Exposez MLflow :   
kubectl port-forward svc/mlflow 5000:5000 
(Dans un nouveau terminal) : 
# Windows PowerShell
$env:MLFLOW_TRACKING_URI="http://localhost:5000"; python app/train.py 
Après l'entraînement, redémarrez le consumer pour qu'il charge le modèle :
kubectl rollout restart deployment consumer
### 6. Observer les résultats 🎉
Lancez l'affichage des logs du consumer :
kubectl logs deployment/consumer -f
------------
### 🛠️ Corrections et Améliorations Apportées (Troubleshooting):
Lors du déploiement, plusieurs défis techniques ont été résolus pour garantir la stabilité du système :

Sécurité MLflow (Invalid Host header) :

Problème : MLflow bloquait les requêtes venant d'autres pods Kubernetes.

Solution : Ajout de l'argument --allowed-hosts "*" dans la commande de démarrage de MLflow dans k8s-deployment.yaml.

Sérialisation Scikit-learn (skops_trusted_types) :

Problème : MLflow refusait d'enregistrer le modèle RandomForest par sécurité.

Solution : Ajout du paramètre skops_trusted_types=["sklearn.tree._tree.Tree"] dans l'appel mlflow.sklearn.log_model du fichier app/train.py.

Simulation de Fraude Autonome :

Problème : Le simulateur ne générait que des transactions normales.

Solution : Modification de la fonction generate_transaction dans app/main.py pour injecter une transaction de 15000$ avec 20% de probabilité, permettant de tester la détection sans passer par Swagger.
-----------------
### 🧰 Stack Technique
**Kubernetes (Docker Desktop)** : Orchestration

**Apache Kafka** : Streaming d'événements

**MLflow** : Model Registry & Tracking

**Feast & Redis** : Feature Store (Online/Offline)

**FastAPI** : API & Simulateur

**Scikit-learn** : Machine Learning (RandomForest)
--------------------------
