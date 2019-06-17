## **Setup database and confirm connectivity from K8S/EKS**

We are using our existing RDS Cluster, and security group rules are already exists for allowing connectivity from EKS cluster to RDS Cluster.

**1. Create Database**

```
CREATE DATABASE stan;
CREATE USER natsstanuser WITH ENCRYPTED PASSWORD 'xxxxxxxxxxx';
GRANT ALL PRIVILEGES ON DATABASE stan TO natsstanuser;
```

**2. Confirm connectivity**
```
psql \
--host=xxxx-pdb-001.xxxxx.us-east-1.rds.amazonaws.com \
--port=5432   \
--username natsstanuser   \
--password   \
--dbname=stan
```

**3. Configure / Restore stan database**

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