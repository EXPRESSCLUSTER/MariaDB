MARIADB with EXPRESSCLUSTER X on Linux
===

About this guide
---
This guide describes how to setup Mariadb with EXPRESSCLUSTER X. 
For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html) .


Configuration
---
In this setup, create 2 nodes (Node1 and Node2 as below) mirror disk type cluster.
Achieving Mariadb high availability By using EXPRESSCLUSTER X. 

### Software versions
- Red Hat Enterprise Linux release 9.0 (Plow)
- Mariadb 11.0(internal version:11.0.3)            
- EXPRESSCLUSTER X 5.1 for Linux
- EXPRESSCLUSTER X Replicator for Linux
- EXPRESSCLUSTER X Database Agent for Linux

### Cluster configurations
- Group resources
  - exec resource
  - floating IP resource
  - mirror disk resource
  
### Monitor resources
  - floating IP resource
  - mirror disk connect monitor resource
  - mirror disk monitor resource
  - mariadb monitor resource

Mariadb setup
---
Please note that the following points are different if you set Mariadb to EXPRESSCLUSTER.
- database have to create Mirror disk that managed by EXPRESSCLUSTER.
  You must set only active server if you create database and database cluster.


Procedure
---
1. EXPRESSCLUSTER setup  

    - We assume the following 2node cluster and explain it.

    ### cluster information
    ||Node1(Active)|Node2(Standby)|
    |---|---|---|
    |Server name|Server1|Server2|
    |IPaddress|10.0.7.131|10.0.7.132|  
    |cluster partition|/dev/sdc1|/dev/sdc1|
    |data partition|/dev/sdb1|/dev/sdb1|
    
    ### failover group information  
    |parameter|value|
    |---|---|
    |name|failover1|
    |Startup Server| Server1 -> Server2 |
    |floating ip address|10.0.7.133|
    |mirror disk resource (mount point))|/database|
    
    - In Config mode of the Cluster WebUI, add failover group to use Mariadb.  
      You need the following resources.
      - Floating ip resource  
      - Mirror disk resource
    
     If want to know how to add the resource, please refer to "EXPRESSCLUSTER X 4.1/2/3 for Linux Installation and Configuration Guide". 
     After you add failover group and execute apply the configuration file, you start failover group by server1.  
     
2. Install Mariadb on both servers 

     Note :- Download Mariadb Community Server for RHEL9 from: [This site](https://dlm.mariadb.com/3405904/MariaDB/mariadb-11.1.2/yum/centos/mariadb-11.1.2-rhel-7-x86_64-rpms.tar) 

    - Unzip Downloaded Tar File **mariadb-11.0.3-rhel-9-x86_64-rpms.tar**
      ```
      [root@rhel131 Downloads]# sudo tar -xvf mariadb-11.0.3-rhel-9-x86_64-rpms.tar
      ```
    - Go to the Extracted Directory path and Set executable permission on setup_repository file 
      
      ```
      [root@rhel131 /]# cd mariadb-11.0.3-rhel-9-x86_64-rpms/
      [root@rhel131 mariadb-11.0.3-rhel-9-x86_64-rpms]# chmod +x setup_repository
      ```

    - By Executing the Setup_repository Script, this will add Mariadb repository on "/etc/yum.repos.d/mariadb.repo"

      ```
      [root@rhel131 mariadb-11.0.3-rhel-9-x86_64-rpms]# ./setup_repository
      ```

    - After adding Mariadb Repository install Mariadb with Yum utility

      ```
      [root@rhel131 /]# yum install mariaDB-mariadb-server.x86_64
      [root@rhel131 /]# systemctl start Mariadb.service
      [root@rhel131 /]# systemctl status Mariadb.service
      ```

    - Configure the root account
      ```
      [root@rhel131 /]# mysql_secure_installation
      ```   
        
3. Mariadb Configuration for Mirror disk (Node1)

    - Create the database directory and mount it with data partition /dev/sdb1
      ```
      [root@rhel131 /]# mkdir database
      [root@rhel131 /]# mount /dev/sdb1 /database
      ```
    - Coping Mariadb data from default location to Mirror Disk.
    
      - systemctl status mariadb
      - systemctl stop mariadb
      - systemctl disable mariadb
      - sudo rsync -av /var/lib/mysql /database
      
4. Perform the below steps on both the Nodes.
  
        
      - Configure the Mariadb Configuration file (/etc/my.cnf.d/mysql-server.cnf) with your favourite editor
          > [mariadb]

          > datadir=/database/mysql
          
          > socket=/database/mysql/mysql.sock  


      - Configure the MySQL Client Configuration file (/etc/my.cnf.d/client.cnf).
          > [client]

          > port=3306
          
          > socket=/database/mysql/mysql.sock
        
        - No other configuration required.        
     
5. Mariadb Setup (Node1)
        
    - Run the Maridb service.
      ```
      [root@rhel131 /]# systemctl start mariadb
      ``` 

    - Create database db_test
      ```
      [root@rhel131 /]# - sudo mariadb

      MariaDB [(none)]> CREATE database db_test;
      ``` 
    - Create user and password for executing database.
      ```
      MariaDB [(none)]> CREATE USER 'test_user'@'localhost' IDENTIFIED BY 'aaaaaa'; 
      ``` 
    - Grant permissions and Apply the Configuration file.
      ```
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON db_test.* TO 'test_user'@'localhost';
      MariaDB [(none)]>  FLUSH PRIVILEGES;
      ```
    - Create Table User
      ```
      MariaDB [(none)]> use db_test;
      MariaDB [(none)]> create table user(id int, name varchar(10));
      MariaDB [(none)]> create index id_index on user(id);
      MariaDB [(none)]> insert into user values(1, 'Naruto');
      MariaDB [(none)]> insert into user values(2, 'Uzumaki');
      ```
    - Confirm the created table
      ```
      MariaDB [db_test]> select * from user;
      +------+---------+
      | id   | name    |
      +------+---------+
      |    1 | Harsh   |
      |    2 | uzumaki |
      +------+---------+
      2 rows in set (0.001 sec)
      ```
    - Stop Mariadb service and umount /dev/sdb1 data partition
      ```
      [root@rhel131 /]# systemctl stop mariadb
      [root@rhel131 /]# umount /database
      ```   
6. MySQL Setup (Node2)

    You must move failover group on the Node2. Configure Mariadb on the Node2.
    - Start Mariadb service
      ```
      [root@rhel131 /]# systemctl start mariadb
      ```
    - Confirm the created Database
      ```
      [root@rhel131 /]# - sudo mariadb
      MariaDB [(none)]> show databases;
      +--------------------+
      | Database           |
      +--------------------+
      | db_test            |
      | information_schema |
      | mysql              |
      | performance_schema |
      | sys                |
      +--------------------+
      7 rows in set (0.001 sec)
      ``` 
    - Confirm the created table
      ```
      MariaDB [db_test]> use db_test;
      MariaDB [db_test]> select * from user;
      +------+---------+
      | id   | name    |
      +------+---------+
      |    1 | Harsh   |
      |    2 | uzumaki |
      +------+---------+
      2 rows in set (0.001 sec)
      ```
    - Stop Mariadb service
      ```
      [root@rhel131 /]# systemctl stop mariadb
      ```

 7. Configure the EXPRESSCLUSTER
 
      - Add the exec resource and configure. 
       
        In Config mode of the Cluster WebUI, Add the exec resource to control MySQL.  
          - Configure the start.sh and stop.sh
            -  In the case of start.sh -> Immediately after "$CLP_DISK" = "SUCCESS", add the "systemctl start mariadb.service"
            -  In the case of stop.sh  -> Immediately after "$CLP_DISK" = "SUCCESS", add the "systemctl stop mariadb.service"
           
      - Add the Mariadb monitor resource
          - Configure the following parameters

              |parameter|value|
              |---|---|
              |Monitor(common) > Monitor Timing > Target Resource|setting the exec resource name|
              |Monitor(special) > Database Name|db_test|
              |Monitor(special) > User Name|test_user|
              |Monitor(special) > password|aaaaaa|
              |Monitor(special) > Library Path|/usr/lib64/mysql/libmariadbd.so.19|
              
      - In Config mode of the Cluster WebUI, execute Apply the Configuration File.
      

8. Verification

    - Confirm that we can access the database where it failover group is running.
