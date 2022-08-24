# Trino-Hive-Postgres-Minio Exercise

This is a personal exercise for me to understand Trino and its architecture, especially with 
Hive Metastore/ Postgres/ Minio. The idea of the exercise is originated from 
https://trino.io/blog/2020/10/20/intro-to-hive-connector.html. The only difference is that I 
changed the DB of Hive Metastore from MariaDB to PostgresDB and tried to implement it 
with k8s with the help of https://github.com/alexcpn/presto_in_kubernetes.

## Architechure Diagram


## Run Locally with Docker-compose
The implementation with docker-compose is referred to https://github.com/bitsondatadev/trino-getting-started/tree/main/hive/trino-minio 
with the help of https://github.com/bitsondatadev/hive-metastore/pull/2/commits to build a Hive Metastore image to
support PostgresDB.

Step 0 - Go to the docker-compose directory
```bash
cd docker-compose
```
Step 1 - Build a Hive Metastore image with the Dockerfile to support PostgresDB
```bash
docker build -t my-hive-metastore .
```
Step 2 - Implement with docker-compose
```bash
docker-compose up -d
```
Step 3 - Create Bucket in MinIO
- account: minio
- password: minio123

Step 4 - Into the runnung trino container
```bash
docker container exec -it docker-compose_trino-coordinator_1 trino
```
Step 5 -  Create schema and table and play around with trino, you can see the trino dashboard from localhost:8080.
```sql
CREATE SCHEMA minio.test
WITH (location = 's3a://test/');

CREATE TABLE minio.test.customer
WITH (
    format = 'ORC',
    external_location = 's3a://test/customer/'
) 
AS SELECT * FROM tpch.tiny.customer;
```

			
### (Optional: see the metadata store in Postgres)
Step 6 - Into the running postgres container
```bash 
docker exec -it "docker-compose_postgres_1" psql -U admin -d "hive_db"
```
Step 7 - Run SQL commands on postgresDB to see where the metadata is stored. 
```sql
SELECT
 "DB_ID",
 "DB_LOCATION_URI",
 "NAME", 
 "OWNER_NAME",
 "OWNER_TYPE",
 "CTLG_NAME"
FROM "DBS";
```

For more metadata detail, kindly check: 
https://github.com/bitsondatadev/trino-getting-started/tree/main/hive/trino-minio

Step 8 - Close down the running containers
```bash
docker-compose down
```

## Run Locally with Kubernetes (with Kind to run locally)
### Prerequisites: Kubectl/ Kind/ Helm
- Kubectl(https://kubernetes.io/docs/tasks/tools/): or with Mac simply try: brew install kubectl
- Kind(https://kind.sigs.k8s.io/): or with Mac simply try: brew install kind
- Helm(https://helm.sh/) or with Mac simply try: brew install helm
 
### Steps
Step 0 - Go to the kubernetes directory and set up Kubernetes
```bash
cd kubernetes
kind create cluster --config kind-cluster-config.yaml
```

Step 1 - set up Minio

- For the first time, you may need to run below command to add minio at first:
```  
helm repo add minio https://charts.min.io/
```
- Install Minio via Helm:
```
helm install minio-test -f minio/values.yaml  minio/minio
```
- Port-forward:
```
kubectl port-forward svc/minio-test-console 9001
```
- Create a bucket test from the UI:
Step 2 - Set up Postgres
- Apply Postgres in Kubernetes with Kubegres:
```
kubectl apply -f https://raw.githubusercontent.com/reactive-tech/kubegres/v1.15/kubegres.yaml
kubectl apply -f postgres/postgres-secret.yaml
kubectl apply -f postgres/kubegres-porstrescluster.yaml
```
- Manually Create a DB after installing Postgres with password: postgresSuperUserPsw
```
kubectl exec -it mypostgres-1-0 —- /bin/sh
psql -U postgres
```
```sql
CREATE DATABASE metadata;
```
Step 3 - Set up Hive Metastore
```
kubectl apply -f hive/hive-initschema.yaml
kubectl create secret generic my-s3-keys --from-literal=access-key=’minio’ --from-literal=secret-key=’minio123’
```
- Change the endpoint IP in metastore-cfg.yaml if needed
```
kubectl get ep | grep mini
<property>
    <name>fs.s3a.endpoint</name>
    <value>http://10.244.1.10:9000</value>
</property>	
```
- Build Hive meta store image if first time
```
docker build -t hivemetastore:3.1.3.5 -f hive/Dockerfile ./hive
```
- Apply hive metastore configuration and create it
``` 
kubectl apply -f hive/metastore-cfg.yaml
kubectl delete -f hive/hive-meta-store-standalone.yaml 
kubectl create -f hive/hive-meta-store-standalone.yaml
```
Step 4 - Set up Trino
```
kubectl apply -f trino/trino_cfg.yaml
kubectl apply -f trino/trino.yaml
kubectl port-forward svc/trino 8080 
```

Step 5 - Access Trino CLI and play around with it, you can see the trino dashboard from localhost:8080.
```
kubectl exec -it trino-cli -- /bin/bash 
trino --server trino:8080 --catalog hive --schema default
```
```sql
SHOW schemas FROM tpcds;
SHOW tables  FROM tpcds.tiny;

CREATE SCHEMA hive.tpcds WITH (location = 's3a://test/warehouse/tpcds/');
CREATE TABLE tpcds.store_sales AS SELECT * FROM tpcds.tiny.store_sales;
SELECT count(*) FROM tpcds.store_sales;
```
Step 6 - Delete Kind cluster
```
kind delete cluster
```