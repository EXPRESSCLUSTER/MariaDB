# Galera Cluster on Kubernetes

## Index
- [Evaluation Environment](#evaluation-environment)
- [Prerequisite](#prerequisite)
- [Deploy MariaDB Galera Cluster on Rook-Ceph](#deploy-mariadb-galera-cluster-on-rook-ceph)
<!--
- [Deploy MariaDB Galera Cluster on NFS](#deploy-mariadb-galera-cluster-on-nfs)
-->
## Evaluation Environment
### Rook-Ceph
- Each worker node of kubernetes has 40GB disk (e.g. /dev/sdb) for Ceph.
  ```
  +-------------------------------------------+
  | Windows Server                            | 
  | - Hyper-V                                 |
  | +------------------------+                |
  | | Master (Control Plane) |                |
  | | Ubuntu 20.04 LTS       |                |
  | | kubernetes 1.18.3      |                |
  | | Docker 19.03.10        |                |
  | +------+-----------------+                |
  |        |                                  |
  | +------+-------------+  +----------+      |
  | | Worker #1, #2, #3  |  |          |      |
  | | Ubuntu 20.04 LTS   +--+ /dev/sdb |      |
  | | kubernetes 1.18.3  |  |     40GB |      |
  | | Docker 19.03.10    |  |          |      |
  | +--------------------+  +----------+      |
  +-------------------------------------------+
  ```
<!--
### NFS
- Install NFS server on another machine. In my case, I have installed NFS server on host OS (CentOS).
  ```
  +-------------------------------------------+
  | CentOS Linux release 7.8.2003 (Core)      | 
  | - KVM                                     |
  | +------------------------+                |
  | | Master (Control Plane) |                |
  | | Ubuntu 20.04 LTS       |                |
  | | kubernetes 1.18.3      |                |
  | | Docker 19.03.10        |                |
  | +------+-----------------+                |
  |        |                                  |
  | +------+-------------+  +------------+    |
  | | Worker #1, #2, #3  |  |            |    |
  | | Ubuntu 20.04 LTS   +--+ NFS Server |    |
  | | kubernetes 1.18.3  |  | on Host OS |    |
  | | Docker 19.03.10    |  |            |    |
  | +--------------------+  +------------+    |
  +-------------------------------------------+
  ```
-->
## Prerequisite
- You need to install Helm. Please refer to [Setup Helm](https://github.com/EXPRESSCLUSTER/Helm/blob/master/SetupHelm.md).
- You need to prepare some provision storage. In this article, we have used [Rook-Ceph](https://github.com/EXPRESSCLUSTER/Rook/blob/master/Rook-Ceph.md).
- If you have the proxy server, you need to run the following command.
  ```sh
  $ export http_proxy=<your proxy server>
  $ export no_proxy=<your local IP address>
  ```

## Deploy MariaDB Galera Cluster on Rook-Ceph
1. Add bitnami repository.
   ```sh
   $ helm repo add bitnami https://charts.bitnami.com/bitnami
   "bitnami" has been added to your repositories
   $ helm repo list
   NAME    URL
   bitnami https://charts.bitnami.com/bitnami
   ```
1. Clone bitnami repository.
   ```sh
   $ git clone https://github.com/bitnami/charts.git
   ```
1. Move to the following directory.
   ```sh
   $ cd charts/bitnami/mariadb-galera
   ```
1. Check the storage class name (e.g. rook-ceph-block).
   ```sh
   $ kubectl get sc
   NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   3d4h
   ```
1. Edit values.yaml as below to use Ceph.
   ```yaml
   image:
     registry: docker.io
     repository: bitnami/mariadb-galera
     tag: 10.4.13-debian-10-r16
   (snip)
     debug: true
   (snip)
   extraEnvVars:
     - name: MARIADB_INIT_SLEEP_TIME
       value: "1800"
   (snip)
   rootUser:
     password: <your password>
     forcePassword: true
   (snip)
   db:
     user: <your user name>
     password: <your password>
     name: my_database
     forcePassword: true
   (snip)
   galera:
     name: galera
   (snip)
     mariabackup:
       user: mariabackup
       password: <your passowrd>
       forcePassword: true
   (snip)
   persistence:
     enabled: true
     mountPath: /bitnami/mariadb
     storageClass: rook-ceph-block
   (snip)
   livenessProbe:
     enabled: true
     initialDelaySeconds: 2400
     periodSeconds: 10
     timeoutSeconds: 1
     successThreshold: 1
     failureThreshold: 3
   readinessProbe:
     enabled: true
     initialDelaySeconds: 2100
     periodSeconds: 10
     timeoutSeconds: 1
     successThreshold: 1
     failureThreshold: 3
   (snip)
   ```
1. Install mariadb-galera.
   ```sh
   $ helm install my-release -f ./values.yaml bitnami/mariadb-galera
   ```
1. Check the all pods are running.
   ```sh
   $ kubectl get pod
   NAME                          READY   STATUS    RESTARTS   AGE
   my-release-mariadb-galera-0   1/1     Running   0          151m
   my-release-mariadb-galera-1   1/1     Running   0          115m
   my-release-mariadb-galera-2   1/1     Running   0          79m   
   ```
<!--
## Deploy MariaDB Galera Cluster on NFS
1. Add bitnami repository.
   ```sh
   $ helm repo add bitnami https://charts.bitnami.com/bitnami
   "bitnami" has been added to your repositories
   $ helm repo list
   NAME    URL
   bitnami https://charts.bitnami.com/bitnami
   ```
1. Clone bitnami repository.
   ```sh
   $ git clone https://github.com/bitnami/charts.git
   ```
1. Move to the following directory.
   ```sh
   $ cd charts/bitnami/mariadb-galera
   ```
1. Create three persistent volumes and check they are available.
   ```sh
   $ kubectl get pv
   NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   pv060   8Gi        RWO            Recycle          Available                                   5s
   pv061   8Gi        RWO            Recycle          Available                                   5s
   pv062   8Gi        RWO            Recycle          Available                                   5s
   ```
1. Edit values.yaml as below to use NFS.
   ```yaml
   image:
     registry: docker.io
     repository: bitnami/mariadb-galera
     tag: 10.4.13-debian-10-r16
   (snip)
     debug: true
   (snip)
   extraEnvVars:
     - name: MARIADB_INIT_SLEEP_TIME
       value: "1800"
   (snip)
   securityContext:
     enabled: true
     fsGroup: 0
     runAsUser: 0
   (snip)
   rootUser:
     password: <your password>
     forcePassword: true
   (snip)
   db:
     user: <your user name>
     password: <your password>
     name: my_database
     forcePassword: true
   (snip)
   galera:
     name: galera
   (snip)
     mariabackup:
       user: mariabackup
       password: <your passowrd>
       forcePassword: true
   (snip)
   persistence:
     enabled: true
     mountPath: /bitnami/mariadb
     storageClass: "-"
   (snip)
   livenessProbe:
     enabled: true
     initialDelaySeconds: 2400
     periodSeconds: 10
     timeoutSeconds: 1
     successThreshold: 1
     failureThreshold: 3
   readinessProbe:
     enabled: true
     initialDelaySeconds: 2100
     periodSeconds: 10
     timeoutSeconds: 1
     successThreshold: 1
     failureThreshold: 3
   (snip)
   ```
   - MARIADB_INIT_SLEEP_TIME
     - On my environment, readiness probe detects error a lot so that set longer value.
   - securityContext
     - In an ideal case, use non-root user but I cannot find out to use non-root user for NFS. I set ```0``` to ```fsGroup``` and ```runAsUser```.
1. Install mariadb-galera.
   ```sh
   $ helm install my-release -f ./values.yaml bitnami/mariadb-galera
   ```
-->