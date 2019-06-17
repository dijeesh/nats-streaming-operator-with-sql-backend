## **Introduction**

## NATS-IO

NATS.io is a simple, secure and high performance open source messaging system for cloud native applications, IoT messaging, and microservices architectures.


### **nats server and nats-streaming-server**

NATS is available in two interoperable modules, the core NATS platform - the NATS server (executable name is gnatsd) referred to simply as NATS and NATS Streaming (executable name is nats-streaming-server)

- Basic NATS Server is designed for high performance and simplicity.
- NATS Server doesn’t provide a persistent store for the messages that you publish over the NATS.
- NATS Streaming comes with a persistent store for having a log for the messages that publish over the NATS server
- NATS Streaming is not a separate server.
- NATS Streaming embeds a NATS server as the messaging server, and provides an extra capability of having a persist logs to be used for event streaming systems.


If you need persistent messaging and delivery guarantees, you can use NATS Streaming instead of the core NATS platform


### **nats-operator**

https://github.com/nats-io/nats-operator

- nats-operator manages NATS clusters atop Kubernetes
- nats-operator automates the creation and administration of Nats cluster
- nats-operator provides a NatsCluster Custom Resources Definition (CRD) that models a NATS cluster
    - This CRD allows for specifying the desired size and version for a NATS cluster, as well as several other advanced options


### **nats-streaming-operator** 

https://github.com/nats-io/nats-streaming-operator

- nats-streaming-operator makes available a NatsStreamingCluster Custom Resources Definition that creates a NATS Streaming Cluster on top of a K8S Cluster.
- nats-streaming-operator can also be used to manage instances backed by a SQL store. In this mode, only a single replica will be created
  - To use DB store support it is needed to include the DB credentials within the NATS Streaming configuration and mount it as a volume.

------------------

## **Setup**

Here are the steps we have performed to set up a nats cluster with nats-streaming-operator backed by a SQL store. We are setting up this cluster in EKS and using RDS as Postgres Backend.

### **1. Clone Repository**

```
git clone https://github.com/dijeesh/nats-streaming-operator-with-sql-backend.git
```
------

### **2. Create nats-io namespace** 

``` 
kubectl create ns nats-io
```
------

### **3. Setup database and confirm connectivity from K8S/EKS**

We are using our existing RDS Cluster, and security group rules are already exists for allowing connectivity from EKS cluster to RDS Cluster. 

**Create Database**

```
CREATE DATABASE stan;
CREATE USER natsstanuser WITH ENCRYPTED PASSWORD 'xxxxxxxxxxx';
GRANT ALL PRIVILEGES ON DATABASE stan TO natsstanuser;
```

**Confirm connectivity**
```
psql \
--host=xxxx-pdb-001.xxxxx.us-east-1.rds.amazonaws.com \
--port=5432   \
--username natsstanuser   \
--password   \
--dbname=stan
```

**Configure / Restore stan database**

Ref: https://github.com/nats-io/nats-streaming-server/blob/master/postgres.db.sql

```
CREATE TABLE IF NOT EXISTS ServerInfo (uniquerow INTEGER DEFAULT 1, id VARCHAR(1024), proto BYTEA, version INTEGER, PRIMARY KEY (uniquerow));

CREATE TABLE IF NOT EXISTS Clients (id VARCHAR(1024), hbinbox TEXT, PRIMARY KEY (id));

CREATE TABLE IF NOT EXISTS Channels (id INTEGER, name VARCHAR(1024) NOT NULL, maxseq BIGINT DEFAULT 0, maxmsgs INTEGER DEFAULT 0, maxbytes BIGINT DEFAULT 0, maxage BIGINT DEFAULT 0, deleted BOOL DEFAULT FALSE, PRIMARY KEY (id));

CREATE INDEX Idx_ChannelsName ON Channels (name(256));

CREATE TABLE IF NOT EXISTS Messages (id INTEGER, seq BIGINT, timestamp BIGINT, size INTEGER, data BYTEA, CONSTRAINT PK_MsgKey PRIMARY KEY(id, seq));

CREATE INDEX Idx_MsgsTimestamp ON Messages (timestamp);

CREATE TABLE IF NOT EXISTS Subscriptions (id INTEGER, subid BIGINT, lastsent BIGINT DEFAULT 0, proto BYTEA, deleted BOOL DEFAULT FALSE, CONSTRAINT PK_SubKey PRIMARY KEY(id, subid));

CREATE TABLE IF NOT EXISTS SubsPending (subid BIGINT, row BIGINT, seq BIGINT DEFAULT 0, lastsent BIGINT DEFAULT 0, pending BYTEA, acks BYTEA, CONSTRAINT PK_MsgPendingKey PRIMARY KEY(subid, row));

CREATE INDEX Idx_SubsPendingSeq ON SubsPending (seq);

CREATE TABLE IF NOT EXISTS StoreLock (id VARCHAR(30), tick BIGINT DEFAULT 0);

-- Updates for 0.10.0
ALTER TABLE Clients ADD proto BYTEA;
```

or 

```
wget https://github.com/nats-io/nats-streaming-server/blob/master/postgres.db.sql -O stan.sql

psql --host=xxxxx-pgdb-001.xxxxx.us-east-1.rds.amazonaws.com --port=5432   --username natsstanuser   --password stan < stan.sql
```

------
### **4. Create secrets** 

```
cd postgres

kubectl -n nats-io create secret generic stan-secret --from-file secret.conf
```

------
### **5. Generate SSL Certificates**

**Install cfsssl**

We will be using cfssl to generate SSL Certificates. Run following commands to install cfssl.

```
sudo curl https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
sudo curl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
sudo chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

**Update Configuration files**

```
cd nats-streaming-operator-with-db-backend/certs
```

Edit route.json and server.json files and modify the cluster details.

**Generate SSL Certificates**

```
# CA Certificates

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

# Server Certificates

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server

# Route Certificates

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=route route.json | cfssljson -bare route

# Client Certificates

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
```

Ref: https://gist.github.com/wallyqs/696b81427df7c239fb34946eb1ae9f92

------
### **6. Add SSL Certificates to EKS/K8S as Secrets**

Make sure you are running following commands from the certs directory

```
kubectl -n nats-io create secret generic nats-clients-tls --from-file ca.pem --from-file client-key.pem --from-file client.pem --from-file server-key.pem --from-file server.pem

kubectl -n nats-io create secret generic nats-routes-tls --from-file ca.pem --from-file route-key.pem --from-file route.pem --from-file server-key.pem --from-file server.pem
```

### **7. Deploy nats-operator Cluster**

```
cd deploy

kubectl apply -f 101_nats_operator_service_account.yaml
kubectl apply -f 102_nats_operator_service_role.yaml
kubectl apply -f 103_nats_operator_deployment.yaml
kubectl apply -f 104_nats_operator_service.yaml
```

Edit 103_nats_operator_deployment.yaml and make sure you are using latest [release](https://github.com/nats-io/nats-operator/releases) before applying.    


### **8. Deploy nats-streaming-operator**

```
cd deploy

kubectl apply -f 201_nats_streaming_operator_service_account.yaml
kubectl apply -f 202_nats_streaming_operator_service_role.yaml
kubectl apply -f 203_nats_streaming_operator_deployment.yaml
kubectl apply -f 204_nats_streaming_operator_service.yaml
```

Edit 203_nats_streaming_operator_deployment.yaml and make sure you are using latest [release](https://github.com/nats-io/nats-streaming-operator/releases) before applying.  

### **9 Verify Cluster resources**

```
kubectl get crd | grep nats
kubectl -n nats-io get pods
kubectl -n nats-io get svc
```

### **10 Exposing service to other namespaces**

Edit 301_other_namespace_svc.yaml, change the namespace and cluster details.

```
cd deploy
kubectl apply -f 301_other_namespace_svc.yaml
```
Now the NATS Cluster Service in nats-io namespace can be directly accessable from your appnamespace.


### **11 Exposing service to outside EKS Cluster**

If you need to expose the nats service to outside the EKS Cluster, use following snippets.

**With in VPC**

Edit 302_expose_via_internal_elb.yaml, change VPC CIDR/Subnet, and cluster details.

```
cd deploy
kubectl apply -f 302_expose_via_internal_elb.yaml
```
This will create an Amazon ELB (Internal) and expose it on por 4222. Access restricted to VPC CIDR Only.


**Public Network**

Edit 303_expose_via_external_elb.yaml, IP Whitelist, and cluster details.

```
cd deploy
kubectl apply -f 302_expose_via_external_elb.yaml
```
This will create an Amazon ELB and expose it on por 4222. Access restricted to Whitelisted IPs only.


Happy Messaging :)