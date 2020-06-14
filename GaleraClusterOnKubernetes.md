# Galera Cluster on Kubernetes

## Index
- [Evaluation Environment](#evaluation-environment)
- [Prerequisite](#prerequisite)
- [Deploy MariaDB Galera Cluster](#deploy-mariadb-galera-cluster-on-rook-ceph)
- [Reference](#reference)

## Evaluation Environment
### Rook-Ceph
- Each worker node of kubernetes has 40GB disk (e.g. /dev/sdb) for Ceph.
  ```
  +--------------------------------------+
  | Windows Server                       | 
  | - Hyper-V                            |
  | +-------------------+                |
  | | Control Plane     |                |
  | | Ubuntu 20.04 LTS  |                |
  | | kubernetes 1.18.3 |                |
  | | Docker 19.03.10   |                |
  | +------+------------+                |
  |        |                             |
  | +------+------------+  +----------+  |
  | | Node #1, #2, #3   |  |          |  |
  | | Ubuntu 20.04 LTS  +--+ /dev/sdb |  |
  | | kubernetes 1.18.3 |  |     40GB |  |
  | | Docker 19.03.10   |  |          |  |
  | +-------------------+  +----------+  |
  +--------------------------------------+
  ```

### Longhorn
- Install open-iscsi on 3 nodes to deploy Longhorn.
  ```
  +-----------------------+
  | Windows Server        | 
  | - Hyper-V             |
  | +-------------------+ |
  | | Control Plane     | |
  | | Ubuntu 20.04 LTS  | |
  | | kubernetes 1.18.3 | |
  | | Docker 19.03.10   | |
  | +------+------------+ |
  |        |              |
  | +------+------------+ |
  | | Node #1, #2, #3   | |
  | | Ubuntu 20.04 LTS  | |
  | | kubernetes 1.18.3 | |
  | | Docker 19.03.10   | |
  | +-------------------+ |
  +-----------------------+
  ```

## Prerequisite
- You need to install Helm. Please refer to [Setup Helm](https://github.com/EXPRESSCLUSTER/Helm/blob/master/SetupHelm.md).
- You need to prepare some provision storage. 
  - [Rook-Ceph](https://github.com/EXPRESSCLUSTER/Rook/blob/master/Rook-Ceph.md).
  - [Longhorn](https://github.com/EXPRESSCLUSTER/Longhorn/blob/master/InstallLonghorn.md)
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
1. Edit values.yaml as below. StorageClass should be rook-ceph-block, longhorn or other storage class name.
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
## Reference
### CPU and Memory Usage
#### Galera Cluster on Rook-Ceph 
```sh
$ kubectl top node
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ubuntu-205   378m         18%    1563Mi          54%
ubuntu-206   338m         16%    1982Mi          68%
ubuntu-207   372m         18%    2094Mi          72%
ubuntu-208   367m         18%    2113Mi          73%
```
<!--
$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)
my-release-mariadb-galera-0   10m          176Mi
my-release-mariadb-galera-1   9m           206Mi
my-release-mariadb-galera-2   7m           202Mi
-->
#### Galera Cluster on Longhorn
```sh
$ kubectl top node
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ubuntu-205   386m         19%    1510Mi          52%
ubuntu-206   271m         13%    847Mi           29%
ubuntu-207   320m         16%    782Mi           27%
ubuntu-208   350m         17%    930Mi           32%
```