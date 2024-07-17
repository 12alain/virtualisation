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

# Création du Namespace

```
vi wordpress-namespace.yml
```

```
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
```
```
kubectl apply -f wordpress-namespace.yml
```
# Création configmap
```
vi mysql-configmap.yml
```
```
# mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  namespace: wordpress
data:
  MYSQL_DATABASE: ma_db

```
```
kubectl apply -f configmap
```
# Création mysql secrert
```
vi mysql-secret.yml
```
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
data:
  MYSQL_USER: <base64_encoded_user>
  MYSQL_PASSWORD: <base64_encoded_password>
```
Avec  
<base64_encoded_user>= echo 'MYSQL_USER' | base64
<base64_encoded_password>= echo ' MYSQL_PASSWORD' | base64
```
kubectl apply -f mysql-secret.yml
```
# depoyement mysql 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
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
        image: mysql:latest
        env:
          - name: MYSQL_ROOT_PASSWORD
            value: "rootpassword"
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: MYSQL_PASSWORD
          - name: MYSQL_DATABASE
            valueFrom:
              configMapKeyRef:
                name: mysql-configmap
                key: MYSQL_DATABASE
          - name: MYSQL_USER
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: MYSQL_USER
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```
```
kubectl apply -f mysql-deployment.yml
```
verifier si tout marche bien avec les commanddes:
```
kubectl describe deployment -n wordpress
kubect logs deployment -n wordpress
```
# create service mysql 
```
vi mysql-service.yml
```
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```
```
kubectl apply -f mysql-service
```

# create pvc

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
```
kubectl apply -f mysql-service 
```

# WordPress
### 1-ConfigMap pour WordPress :
```
vi wordpress-configmap.yml
```
```
# wordpress-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-configmap
  namespace: wordpress
data:
  WORDPRESS_DB_HOST: mysql
```
```
kubectl apply -f wordpress-configmap.yml
```
### 2-Secret pour WordPress :
```
vi wordpress-secret.yml
```
```
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
  namespace: wordpress
type: Opaque
data:
  MYSQL_USER: dXNlcg==
  MYSQL_PASSWORD: cGFzc2Vy

```
```
kubectl apply -f wordpress-secret.yml
```
### 3-Déploiement WordPress :
```
vi wordpress-deployment.yml
```
```
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
          - name: WORDPRESS_DB_HOST
            valueFrom:
              configMapKeyRef:
                name: wordpress-configmap
                key: WORDPRESS_DB_HOST
          - name: WORDPRESS_DB_USER
            valueFrom:
              secretKeyRef:
                name: wordpress-secret
                key: MYSQL_USER
          - name: WORDPRESS_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: wordpress-secret
                key:  MYSQL_PASSWORD
        ports:
        - containerPort: 80
```
### 4.Service WordPress yaml:

```
vi wordpress-service.yml

```
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
  selector:
    app: wordpress
```
```
kubectl apply -f wordpress-service.yml
```
# PHPMyAdmin
 ### 1. ConfigMap pour PHPMyAdmin :
```
vi phpmyadmin-configmap.yml
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: phpmyadmin-configmap
  namespace: wordpress
data:
  PMA_HOST: mysql
```
```
kubectl apply -f phpmyadmin-configmap.yml
```
### Secret pour PHPMyAdmin :
```
vi phpmyadmin-secret.yml
```
```

apiVersion: v1
kind: Secret
metadata:
  name: phpmyadmin-secret
  namespace: wordpress
type: Opaque
data:
  MYSQL_USER: dXNlcg==
  MYSQL_PASSWORD: cGFzc2Vy

```
```
kubectl apply -f phpmyadmin-secret.yml
```
### Déploiement PHPMyAdmin :

```
vi phpmyadmin-deployment.yml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: phpmyadmin/phpmyadmin:latest
        env:
          - name: PMA_HOST
            valueFrom:
              configMapKeyRef:
                name: phpmyadmin-configmap
                key: PMA_HOST
          - name: PMA_USER
            valueFrom:
              secretKeyRef:
                name: phpmyadmin-secret
                key:  MYSQL_USER
          - name: PMA_PASSWORD
            valueFrom:
              secretKeyRef:
                name: phpmyadmin-secret
                key: MYSQL_PASSWORD
        ports:
        - containerPort: 80
```
```
kubectl apply -f phpmyadmin.yml 
```
### Service PHPMyAdmin :
```
vi phpmyadmin-service.yml
```

```
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin
  namespace: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
  selector:
    app: phpmyadmin
```
```
kubectl apply -f phpmyadmin-service.yml

```
kubectl apply -f frontend-deployment.yml
kubectl apply -f mysql-service.yml
kubectl apply -f backend-service.yml
kubectl apply -f frontend-service.yml
```
