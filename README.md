# Étape 1 : Construire et pousser les images Docker

- Backend (Flask) Dockerfile : backend/Dockerfile

```
# Déplacez-vous dans le répertoire backend

cd ml_project/backend
docker build -t your_dockerhub_username/backend:latest .
docker push your_dockerhub_username/backend:latest

```
- Frontend (React) Dockerfile : frontend/Dockerfile

```
# Déplacez-vous dans le répertoire frontend
cd ../frontend
docker build -t your_dockerhub_username/frontend:latest .
docker push your_dockerhub_username/frontend:latest

```
# Étape 2 : Créer un Namespace Kubernetes
Créez un fichier namespace.yml :
```
apiVersion: v1
kind: Namespace
metadata:
  name: ml-project
```
Appliquez ce namespace :
```
kubectl apply -f namespace.yml

```
# Étape 3 : Créer des fichiers de déploiement Kubernetes
### a. Déploiement pour MySQL
 Créez un fichier mysql-deployment.yml :
 ```
apiVersion: v1
kind: Deployment
metadata:
  name: mysql
  namespace: ml-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mysql-config
              key: MYSQL_DATABASE
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-persistent-storage
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc


 ```

### b. Déploiement pour l’application Backend
Créez un fichier backend-deployment.yml :
 ```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ml-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your_dockerhub_username/backend:latest
        env:
        - name: MYSQL_HOST
          value: "mysql"
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: MYSQL_DATABASE
        - name: FLASK_ENV
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: FLASK_ENV
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        ports:
        - containerPort: 5000

  ```
### c. Déploiement pour l’application Frontend
Créez un fichier frontend-deployment.yml :


  ```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ml-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: your_dockerhub_username/frontend:latest
        env:
        - name: REACT_APP_BACKEND_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: REACT_APP_BACKEND_URL
        - name: CHOKIDAR_USEPOLLING
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: CHOKIDAR_USEPOLLING
        ports:
        - containerPort: 3000

  ```
  # Étape 4 : Créer les services

  ### a. Service pour MySQL
  Créez un fichier mysql-service.yml :
   ```
   apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: ml-project
spec:
  type: ClusterIP
  ports:
  - port: 3306
  selector:
    app: mysql
 ```
### b. Service pour le Backend
Créez un fichier backend-service.yml :
```
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ml-project
spec:
  type: ClusterIP
  ports:
  - port: 5000
  selector:
    app: backend


```

### c. Service pour le Frontend
Créez un fichier frontend-service.yml :
```
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ml-project
spec:
  type: NodePort
  ports:
  - port: 3000
    nodePort: 30001
  selector:
    app: frontend

```

# Étape 5 : Créer les ConfigMaps
### a. ConfigMap pour MySQL
Créez un fichier mysql-configmap.yml :

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: ml-project
data:
  MYSQL_DATABASE: ml_project

```
### b. ConfigMap pour le Backend
Créez un fichier backend-configmap.yml :

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: ml-project
data:
  MYSQL_DATABASE: ml_project
  FLASK_ENV: development


```
### c. ConfigMap pour le Frontend
Créez un fichier frontend-configmap.yml :
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: ml-project
data:
  REACT_APP_BACKEND_URL: http://backend:5000
  CHOKIDAR_USEPOLLING: "true"


```
# Étape 6 : Créer un Secret
Créez un fichier mysql-secret.yml :
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: ml-project
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: <base64_encoded_root_password>
  MYSQL_USER: <base64_encoded_user>
  MYSQL_PASSWORD: <base64_encoded_password>
```
Utilisez echo -n 'your_password' | base64 pour encoder vos valeurs en base64.


# Étape 7 : Créer un PersistentVolumeClaim
Créez un fichier mysql-pvc.yml :
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: ml-project
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
# Étape 8 : Appliquer les configurations
Appliquez toutes les configurations dans l'ordre :
```
kubectl apply -f namespace.yml
kubectl apply -f mysql-secret.yml
kubectl apply -f mysql-configmap.yml
kubectl apply -f backend-configmap.yml
kubectl apply -f frontend-configmap.yml
kubectl apply -f mysql-pvc.yml
kubectl apply -f mysql-deployment.yml
kubectl apply -f backend-deployment.yml
kubectl apply -f frontend-deployment.yml
kubectl apply -f mysql-service.yml
kubectl apply -f backend-service.yml
kubectl apply -f frontend-service.yml
```
