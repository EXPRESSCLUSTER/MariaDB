MariaDB Database Quick Start Guide for EXPRESSCLUSTER X on Windows
===

About this guide
---
This guide describes how to setup MariaDB with EXPRESSCLUSTER X 5.1.

For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html) .


## **System overview**

### **System requirement**
- 2 servers
  - IP reachable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
    - Cluster partition size depends on ECX version.
    - X 5.1 or later: 1024MB
    - Data partition size depends on Database sizing.
- MariaDB Database are installed on local partition on each server.

### **System configuration**
- Windows Server 2022 Standard
- MariaDB 10.11.4
- EXPRESSCLUSTER X 5.1

        Sample configuration
		<LAN>
		 |  +--------------------------+
		 |  | Primary Server           |
		 |  | - noimariawdb01 (Win2k22)|
		 |  | - MariaDB10.11.4         |
		 |  | - EXPRESSCLUSTER X 5.1   |
		 |  | IP Address:10.0.7.180    |
		 |  | RAM   : 4GB              |
		 |  | Disk 0: 40GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 30GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +-----------+--------------+
		 |              |
		 |              | Mirroring
		 |              |
		 |  +-----------+--------------+
		 |  | Secondary Server         |
		 |  | - noimariawdb02 (Win2k22)|
		 |  | - MariaDB10.11.4         |
		 |  | - EXPRESSCLUSTER X 5.1   |
		 |  | IP Address:10.0.7.181    |
                 |  | RAM   : 4GB              |
		 |  | Disk 0: 40GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 30GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +--------------------------+
		 |

#### Cluster configuration
- Network Partition Resolution resource (PING method) : `pingnp1`

- Failover Group: `MariaDBfailover`
	- Group resource
		- fip                : floating IP address resource
		- md                 : mirror disk resource
		- mariaDB_service    : service resource for MariaDB

    - Monitor resource

	    - fipw1             : floating IP monitor resource
	    - mdw1              : mirror disk monitor resource
	    - servicew1         : service monitor resource for MariaDB_service
	    - userw             : user-mode monitor resource


## **Basic cluster setup on primary and pecondary servers**


 ### **1. Install EXPRESSCLUSTER X (ECX)**
 ### **2. Register ECX licenses**
- EXPRESSCLUSTER X 5.1 for Windows
- EXPRESSCLUSTER X Replicator 5.1 for Windows

### **3. Create a cluster and a failover group On Primary server**
    - Network partition: 
        pingnp1: PING method
    - Group:
        MariaDBfailover
        fip: floating IP resource
        md : mirror disk resource
        
### **4. Start failover groupon primary server**

      
       +----------------------+-----------------------------+
       | Parameter            | Value                       |
       +----------------------+-----------------------------+
       | Name                 | MariaDBFailover             |
       | Startup Server       | noimariawdb01->noimariawdb02|
       | Floating IP Address  | 10.0.7.184                  |
       | Mirror Disk resource | D:\MariaDB                  |
       +----------------------+-----------------------------+

 If want to know how to add the resource, please refer to "EXPRESSCLUSTER X 5.1 for Windows Installation and Configuration Guide". 

 After you add failover group, execute and apply the configuration file, you start failover group by noimariawdb01.  
     
### **5. Install mariaDB on both servers**

- Install MariaDB 10.11.4 on both the servers.
    
    - For installation procedure please refer to [this site](https://www.mariadbtutorial.com/getting-started/install-mariadb/ ).

### Install mariaDB on both servers


1. Login as an user with Administrator privilege and Execute the .msi file on the server.
1. Installation Directory
    - Specify the local directory (C: drive) as **Installation Directory**.
    - e.g. *C:\Program Files\MariaDB 10.4\*
1. Choose setup type
    - Complete
1. Data Directory
    - Specify the local directory (C: drive) as **Data Directory**.
    - e.g. *C:\Program Files\MariaDB 10.4\data*
1. Ready to Install
    - Click **Next**
1. Completing the MariaDB Setup Wizard
    - please keep it checked and click **Finish**.

### **6. Configure mariaDB database path in mirror drive (Node1)**

- Create the database directory in mirror drive e.g. D:\MariaDB
- Stop MariaDB service from services.msc.
- Copy MariaDB data from default directory (C:\Program Files\MariaDB 10.11\data) to Mirror disk (D:\MariaDB)
- Right Click on the newly created folder and make sure that it has all type permissions for local MongoDB system user.      

### **7. Perform below steps on both the nodes one by one after group failover move.**
 - Open the configuration file in a text editor and modify the MariaDB configuration file with storage location.
 - Configure the MariaDB Configuration file (C:\Program Files\MariaDB 10.11\data\my.ini).
    > [mysqld]
    > datadir=D:\MariaDB\data
          
### **8. Testing Of mariaDB database on ECX cluster**

- MariaDB Setup (Node1)
        
    - Click on Start Tab and Open MySQL 8.0 Client Line.
      - Enter password:- root@123 (Administrative Credential)

    - Verify the location of database 
         
```
    MariaDB [(none)]> select @@datadir;
    +------------------+
    | @@datadir        |
    +------------------+
    | D:\MariaDB\Data\ |
    +------------------+
    1 row in set (0.00 sec)
```

- Check the MariaDB database


```
    MariaDB [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    4 rows in set (0.144 sec)
```         
 
            
- **Create Database in MySql 8.0 client line for testing database on node1.**

 ```
    MariaDB [(none)]> create database test2;
    Query OK, 1 row affected (0.03 sec)

    MariaDB [(none)]> use test2;
    Database changed

    MariaDB [Test2]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test2              |
    +--------------------+
    5 rows in set (0.14 sec)
```    

- Create table 

``` 
    MariaDB [Test2]> create table demo160(id int, name varchar(10));
    Query OK, 0 rows affected (0.13 sec)

    MariaDB [Test2]> create index id_index on demo160(id);
           Query OK, 0 rows affected (0.17 sec)
           Records: 0  Duplicates: 0  Warnings: 0
```

- Insert the values in the tables

```
    MariaDB [Test2]> insert into demo160 values(1, 'harsh');
            Query OK, 1 row affected (0.06 sec)
    MariaDB [Test2]> insert into demo160 values(2, 'mukesh');
            Query OK, 1 row affected (0.01 sec)

```
- Verify the created table in MariaDB database

```
    MariaDB [test2]> select * from demo160;
           +------+--------+
           | id   | name   |
           +------+--------+
           |    1 | harsh  |
           |    2 | mukesh |
           +------+--------+
           3 rows in set (0.00 sec)
```




### **9. You have to move failover group on the node2 for testing MariaDB database failover.**

         
- Check and Confirm the database on MariaDB which we have created on node1.

```
    MariaDB [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test2              |
    +--------------------+
    5 rows in set (0.14 sec)
```

   - The data is replicated from primary to secondary server successfully.   

    
### **10. Change the startup type of MariaDB service (node1 & node2)**

- Open the Windows Service Manager     
  - On Command Prompt
        > services.msc
- Change the startup type of MariaDB Service to Manual.

### **11. Configure MariaDB Service in ECX Cluster**
 - Add the service resource to control MariaDB and configure in Config mode of the Cluster WebUI.
 - mariadbservice       : service resource for MariaDB Server(MariaDB)
 - Execute Apply the Configuration File in Config mode of the Cluster WebUI.
                 
### **12. Verification**

- Confirm that we can access the database where it failover group is running.
