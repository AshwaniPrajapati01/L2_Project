# PostgreSQL Multi Master-Slave Replication

# **Table of content** {#table-of-content}

**[Table of content	1](#table-of-content)**

[Step 1: Sabhee Master-Slave Containers ko connect kerne ke liye Networks ]

[Step 2: Sabhee Master-Slave Containers Image pull Karo	]

[Step 3 : Postgres Configuration sabhi containers pe ]

[Step 4 : Replication User Banao (Sirf Masters pe) ] 

[Step 5 : Master-Slave Streaming Replication Setup ]

[Step 6 : Master-Master Logical Replication Setup in between MasterA AND MasterB ]

# 


# 

# PostgreSQL Multi Master-Slave Replication

## **Step 1: Sabhee Master-Slave Containers ko connect kerne ke liye Networks** {#step-1:-sabhee-master-slave-containers-ko-connect-kerne-ke-liye-networks}

Network       :   172.18.0.0    
Container\_1 :  172.18.0.101 :-  MASTER A  
Container\_2 :  172.18.0.102 :-  SLAVE A  
Container\_3 :  172.18.0.103 :-  MASTER B  
Container\_4 :  172.18.0.104 :-  SLAVE B

## **Step 2: Sabhee Master-Slave Containers Image pull Karo** {#step-2:-sabhee-master-slave-containers-image-pull-karo}

```
Step 1.1: Master A

podman run -d \
  --network networkA \     
  --ip 172.18.0.101 \
  -p 5432:5432 \
  --name masterA \
  -h masterA \
  -e POSTGRES_DB=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=ok \
  -v /home/ashwani/mydir/data/psql/masterA:/var/lib/postgresql/data \
  docker.io/library/postgres:17

8075cef03ef79b08ffe79fabb93b889b5f299fdc2607b3f10e0f6fa6ddac43ca


Step 1.2: Slave A

podman run -d \
  --network networkA \
  --ip 172.18.0.102 \
  -p 5433:5432 \
  --name slaveA \
  -h slaveA \
  -e POSTGRES_DB=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=ok \
  -v /home/ashwani/mydir/data/psql/slaveA:/var/lib/postgresql/data \
  docker.io/library/postgres:17

eee7565fccafe133976bc6ef7d56e0b21f91053334cd17f328cf0c0eb2313f9e


Step 1.3 Master B

podman run -d \
  --network networkA \
  --ip 172.18.0.103 \
  -p 5434:5432 \
  --name masterB \
  -h masterB \
  -e POSTGRES_DB=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=ok \
  -v /home/ashwani/mydir/data/psql/masterB:/var/lib/postgresql/data \
  docker.io/library/postgres:17

f9c9b13c5fc37f55c43e6b42a2b11462febd38edceecc34712d9a70ce4a2ecd5


Step1.4: SlaveB

podman run -d \
  --network networkA \
  --ip 172.18.0.104 \
  -p 5435:5432 \
  --name slaveB \
  -h slaveB \
  -e POSTGRES_DB=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=ok \
  -v /home/ashwani/mydir/data/psql/slaveB:/var/lib/postgresql/data \
  docker.io/library/postgres:17

d2db3f4610325ebaddf0ebb689dee22b962f1e88f1a8a207204143cafc351c70
```

## **Step 3 : Postgres Configuration sabhi containers pe** {#step-3-:-postgres-configuration-sabhi-containers-pe}

Sabhi containers pe **postgresql.conf** aur **pg\_hba.conf** configure karna hoga.

```
Step 2.1: postgresql.conf file modify karo
listen_addresses = '*'
wal_level = logical
synchronous_commit = on
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on


#ssl = on

```
**Step 3.2: `pg_hba.conf` modify karo MATER A AND MASTER B**
```
host	replication	      replicauser  	172.18.0.101/24  	md5
host	all         	replicator  	172.18.0.102/24     md5
host	all         	replicauser  	172.18.0.103/24  	md5
host	all         	replicator  	172.18.0.104/24  	md5

```

**Step 3.3: `pg_hba.conf` modify karo slave A AND slave B**
```
host	all         	replicauser  	172.18.0.101/24  	md5
host	all         	replicator  	172.18.0.102/24     md5
host	all         	replicauser  	172.18.0.103/24  	md5
host	all         	replicator  	172.18.0.104/24  	md5

```

## **Step 4 : Replication User Banao (Sirf Masters pe)** {#step-4-:-replication-user-banao-(sirf-masters-pe)}

Dono masters (MasterA aur MasterB) pe `replicator` user create karo:

```
CREATE USER replicator REPLICATION LOGIN CONNECTION LIMIT 5 ENCRYPTED PASSWORD 'ok'; 

```

## **Step 5 : Master-Slave Streaming Replication Setup** {#step-5-:-master-slave-streaming-replication-setup}

Yeh step SlaveA aur SlaveB ke liye hai, jo MasterA aur MasterB ke slaves honge.

**Step 5.1: Slave Server Pe Data Copy Karo**

**Pehle SlaveA pe (jo MasterA ka slave hoga):**
```
rm -rf slaveA

pg_basebackup -R -D /var/lib/postgresql/replica -Fp -Xs -v -P -h 172.18.0.101 -p 5432 -U replicauser

podman rm -f slaveA

mv replica slaveA

```

**Phir SlaveB pe (jo MasterB ka slave hoga):**
```
rm -rf slaveB

pg_basebackup -R -D /var/lib/postgresql/replica -Fp -Xs -v -P -h 172.18.0.103 -p 5432 -U replicauser

podman rm -f slaveB

mv replica slaveB

```
**Step 5.2 : Create database in MasterA and MasterB for checking Streaming Replication**
```
CREATE DATABASE ashwani;

/C ashwani

CREATE TABLE test_table (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO test_table (data) VALUES ('Hello from MasterA');

```
**Phir SlaveA AND SlaveB  pe check karo:**
```
SELECT * FROM test_table; 
```

## **Step 6 : Master-Master Logical Replication Setup in between MasterA AND MasterB**  {#step-6-:-master-master-logical-replication-setup-in-between-mastera-and-masterb}

Ab MasterA aur MasterB ke beech logical replication setup karna hoga.

### **Step 6.1: Publications Banao Masters pe**

**MasterA pe:**
```
podman exec -it masterA bash

psql -U postgres

/C ashwani

CREATE PUBLICATION pub_masterA FOR ALL TABLES;

```

**MasterB pe:**
```
podman exec -it masterB bash

psql -U postgres

/C ashwani

CREATE PUBLICATION pub_masterB FOR ALL TABLES;

```
**Step 6.2: Subscriptions Banao**

**MasterA pe:**

```
CREATE SUBSCRIPTION sub_MasterA
CONNECTION 'host=172.18.0.103 dbname=ashwani user=replicator password=redhat'
PUBLICATION pub_masterB
WITH (copy_data = false, origin = 'none');

```
**MasterB pe:**

```
CREATE SUBSCRIPTION sub_MasterB
CONNECTION 'host=172.18.0.101 dbname=ashwani user=replicator password=redhat'
PUBLICATION pub_MasterA
WITH (copy_data = false, origin = 'none');

```
# **Step 6 : Testing Replication**

**Data insert in MasterA table test\_table and check in all containers** 
```

podman exec -it masterA bash
 
psql -U postgres

/C ashwani

INSERT INTO test_table (id, data) VALUES
(7, 'Hello from Master7'),
(8, 'Hello from Master8'),
(9, 'Hello from Master9'),
(10, 'Hello from MasterA0'),
(11, 'Hello from MasterA1');

```
**Data insert in Master B table test\_table and check in all containers** 

```
podman exec -it masterB bash

psql -U postgres 

/C ashwani

INSERT INTO test_table (id, data) VALUES
(12, 'Hello from MasterA2'),
(13, 'Hello from MasterA3'),
(14, 'Hello from MasterA4'),
(15, 'Hello from MasterA5'),
(16, 'Hello from MasterA6');
`


**\---------------------------------------------------------------------------------------------------------------**  
   
      
