

###############################################################################
# Create todo-api docker image
###############################################################################
mkdir /root/todo-api

cd /root/todo-api

wget --no-check-certificate --content-disposition https://github.com/brahimmachkour/katacoda-scenarios/raw/main/k8s-test/assets/todo-api.jar

cat > Dockerfile << EOF
FROM openjdk
	
WORKDIR /app

COPY todo-api.jar /app

EXPOSE 9090

ENV SPRING_DATASOURCE_PASSWORD=
ENV SPRING_DATASOURCE_USER=
ENV SPRING_DATASOURCE_URL=

ENTRYPOINT ["java", "-jar", "/app/todo-api.jar"]
EOF

docker build -t todo-api .
docker tag todo-api:latest bmachkour/todo-api:latest
docker push bmachkour/todo-api:latest

###############################################################################
# Create todo-ihm docker image
###############################################################################
mkdir /root/todo-ihm

cd /root/todo-ihm

wget --no-check-certificate --content-disposition https://github.com/brahimmachkour/katacoda-scenarios/raw/main/k8s-test/assets/todo-ihm.jar

cat > Dockerfile << EOF
FROM openjdk
	
WORKDIR /app

COPY todo-ihm.jar /app

EXPOSE 8080

ENV TODO_API_URL=

ENTRYPOINT ["java", "-jar", "/app/todo-api.jar"]
EOF

docker build -t todo-ihm .
docker tag todo-ihm:latest bmachkour/todo-ihm:latest
docker push bmachkour/todo-ihm:latest

###############################################################################
# Create Service mysql
###############################################################################
mkdir /root/mysql-data

cat > /root/mysql-pv.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /root/mysql-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF

kubectl create -f mysql-pv.yaml

cat > mysql.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
EOF

kubectl create -f mysql.yaml

###############################################################################
# Create Service todo-api
###############################################################################
cat > todo-api.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-api
  labels:
    app: todo-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-api
  template:
    metadata:
      labels:
        app: todo-api
    spec:
      containers:
      - name: todo-api
        image: bmachkour/todo-api:latest
		command: ["java", "-jar", "/app/todo-api.jar"]
        ports:
        - containerPort: 9090
        env:
        - name: SPRING_DATASOURCE_PASSWORD
          value: password
        - name: SPRING_DATASOURCE_USER
          value: root
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql.default.svc.cluster.local:3306/mysql?useSSL=false
---
apiVersion: v1
kind: Service
metadata:
  name: todo-api
spec:
  type: ClusterIP
  selector:
    app: todo-api
  ports:
    - port: 9090
      targetPort: 9090
EOF

kubectl create -f todo-api.yaml

###############################################################################
# Create Service todo-ihm
###############################################################################
cat > todo-ihm.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-ihm
  labels:
    app: todo-ihm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-ihm
  template:
    metadata:
      labels:
        app: todo-ihm
    spec:
      containers:
      - name: todo-ihm
        image: bmachkour/todo-ihm:latest
        ports:
        - containerPort: 9090
        env:
        - name: TODO_API_URL
          value: http://todo-api.default.svc.cluster.local:9090
---
apiVersion: v1
kind: Service
metadata:
  name: todo-ihm
spec:
  type: NodePort
  selector:
    app: todo-ihm
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30003
EOF

kubectl create -f todo-ihm.yaml
