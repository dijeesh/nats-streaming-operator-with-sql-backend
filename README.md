## **Introduction**

## NATS-IO

NATS.io is a simple, secure and high performance open source messaging system for cloud native applications, IoT messaging, and microservices architectures.


### **nats server and nats-streaming-server**

NATS is available in two interoperable modules, the core NATS platform -the NATS server (executable name is gnatsd) referred to simply as NATS and NATS Streaming (executable name is nats-streaming-server)

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

[1. Setup database and confirm connectivity from K8S/EKS](https://github.com/dijeesh/setup/01_setup_database_backend_and_confirm_connectivity)

### **1. Setup database and confirm connectivity from K8S/EKS**

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


### **2. Create nats-io namespace** 

``` 
kubectl create ns nats-io
```

### **3. Create secrets** 

1. 