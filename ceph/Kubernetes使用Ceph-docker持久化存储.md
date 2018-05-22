**Kubernetes使用ceph-docker持久化存储**

目录

[1 部署架构 3](#部署架构)

[1.1 主机 3](#主机)

[1.2 前提 3](#前提)

[2 Ceph集群 4](#ceph集群)

[2.1 前提 4](#前提-1)

[2.2 mon/mgr/mds/rgw 4](#monmgrmdsrgw)

[2.3 osd 6](#osd)

[3 Kubernetes使用ceph 8](#kubernetes使用ceph)

[3.1 前提 8](#前提-2)

[3.1.1 Ceph配置 8](#ceph配置)

[3.1.2 安装ceph client 9](#安装ceph-client)

[3.1.3 Ceph池创建 9](#ceph池创建)

[3.2 mount到本地使用 11](#mount到本地使用)

[3.2.1 mount挂载ceph 11](#mount挂载ceph)

[3.2.2 app使用挂载路径 12](#app使用挂载路径)

[3.3 pv/pvc方式使用 14](#pvpvc方式使用)

[3.3.1 创建pv/pvc 14](#创建pvpvc)

[3.3.2 创建pod 17](#创建pod)

[3.4 StorageClass方式使用 19](#storageclass方式使用)

[3.4.1 创建StorageClass 19](#创建storageclass)

[3.4.2 创建deployment 21](#创建deployment)

[4 附录 22](#附录)

[4.1 参考 22](#参考)

1.  部署架构

    1.  主机

| **主机名**  | **IP**             | **ceph角色**          | **osd设备** | **宿主机目录 (配置和数据)** |
|-------------|--------------------|-----------------------|-------------|-----------------------------|
| node1       | 192.168.120.173/24 | mon,mgr, mds, rgw,osd | /dev/sdb    | /etc/ceph/ /var/lib/ceph/   |
| node2       | 192.168.120.174/24 | osd                   | /dev/sdb    | /etc/ceph/ /var/lib/ceph/   |
| node3       | 192.168.120.172/24 | osd                   | /dev/sdb    | /etc/ceph/ /var/lib/ceph/   |
| k8s-master1 | 192.168.120.176/24 | client                |             |                             |
| k8s-node1   | 192.168.120.175/24 | client                |             |                             |

前提
----

>   部署前在说有主机上进行时钟同步

yum clean all

yum install chrony -y

systemctl enable chronyd.service

systemctl restart chronyd.service

Ceph集群
========

Ceph集群部署到docker里.

前提
----

\#在所有主机上拉取ceph 镜像,此处下载版本为ceph version 12.2.4 luminous,

\#ceph/daemon官方镜像地址<https://hub.docker.com/r/ceph/daemon/>

docker pull ceph/daemon

![](media/9b9ea2a70b90eae916338a2d619d984e.png)

mon/mgr/mds/rgw
---------------

在node1安装mon，mgr，mds和rgw

\#在所有宿主机准备目录

mkdir -p /etc/ceph

mkdir -p /var/lib/ceph/

\#清除未用的container

docker container prune -f

rm -fr /etc/ceph/\*

rm -fr /var/lib/ceph/\*

\# 安装mon，MON_IP就是宿主机的IP地址

docker run -d --net=host --name=mon --restart=always \\

\-v /etc/ceph:/etc/ceph \\

\-v /var/lib/ceph/:/var/lib/ceph/ \\

\-e MON_IP=192.168.120.173 \\

\-e CEPH_PUBLIC_NETWORK=192.168.120.0/24 \\

ceph/daemon mon

\#安装mgr

docker run -d --net=host --name=mgr --restart=always \\

\-v /etc/ceph:/etc/ceph \\

\-v /var/lib/ceph/:/var/lib/ceph/ \\

ceph/daemon mgr

\#安装mds

docker run -d --net=host --name=mds --restart=always \\

\-v /etc/ceph:/etc/ceph \\

\-v /var/lib/ceph/:/var/lib/ceph/ \\

\-e CEPHFS_CREATE=1 \\

ceph/daemon mds

\#安装rgw

docker run -d --net=host --name=rgw --restart=always \\

\-v /var/lib/ceph/:/var/lib/ceph/ \\

\-v /etc/ceph:/etc/ceph \\

ceph/daemon rgw

![](media/bc7ec8452bbd10fedbb4c89e8c70c788.png)

![](media/494431358dd1d1f2fb34b0a7562e1c8a.png)

\#复制配置文件

\#将 node1 上的配置文件复制到 node02 和
node03,复制的路径包含/etc/ceph和\#/var/lib/ceph/bootstrap-\*下的所有内容。

ssh root\@node2 mkdir -p /var/lib/ceph

scp -r /etc/ceph root\@node2:/etc

scp -r /var/lib/ceph/bootstrap\* root\@node2:/var/lib/ceph

ssh root\@node3 mkdir -p /var/lib/ceph

scp -r /etc/ceph root\@node3:/etc

scp -r /var/lib/ceph/bootstrap\* root\@node3:/var/lib/ceph

![](media/e4d64a1c3cfe04c590bec8da6d62c309.png)

osd
---

\#在三台osd主机上分别执行

docker run -d --restart=always \\

\--net=host \\

\-v /etc/ceph:/etc/ceph \\

\-v /var/lib/ceph/:/var/lib/ceph/ \\

\-v /dev/:/dev/ \\

\--privileged=true \\

\-e OSD_FORCE_ZAP=1 \\

\-e OSD_DEVICE=/dev/sdb \\

ceph/daemon osd_ceph_disk

![](media/7fde4c86200791422a4282d4982a39e4.png)

\#在 mon主机上检查集群情况

docker exec mon ceph -s

![](media/7e84040d6f8397f526de49d555a583bd.png)

\#在 osd主机上检查硬盘分区

fdisk -l /dev/sdb

![](media/47daa4ff43ebcab21b073b21b9701b6a.png)

Kubernetes使用ceph
==================

Kubernetes三种方式使用ceph进行持久化：

1.  mount到本地目录使用

2.  Pv/pvc方式持久化存储

3.  Storageclass方式持久化存储

    1.  前提

        1.  Ceph配置

\#ceph配置/etc/ceph拷贝到所有kubernetes主机节点,作为ceph的client使用

scp -r /etc/ceph root\@k8s-master1:/etc

scp -r /etc/ceph root\@k8s-node1:/etc

![](media/7d66f76bb909e2abe5b6afa4a9a394f1.png)

### 安装ceph client

\#kubernetes节点安装ceph客户端

yum install ceph-common -y

\#或指定版本安装

\# yum install ceph-common-12.2.4

![](media/bf8dc05d537fb9e831b322aadda65f23.png)

### Ceph池创建

在执行创建该 Pod 之前，先在ceph client主机手动创建pool k8s-pool和 image
foo，后续用到。

\#ceph 集群创建一个新的存储池k8s-pool， Placement Group的个数为64

ceph osd pool create k8s-pool 64

\#在ceph池k8s-pool中创建一个块设备 image镜像 foo

rbd create --size 1G k8s-pool/foo -m node1 --image-format 2 --image-feature
layering

\# map映射到本地

rbd map k8s-pool/foo --name client.admin -m node1 -k
/etc/ceph/ceph.client.admin.keyring

\#格式化块设备

mkfs.xfs /dev/rbd0

\#指定pool应用类型为rbd

ceph osd pool application enable k8s-pool rbd

![](media/3a99aa49750568189cc04126caf8e548.png)

\#查看已经映射的Block Device信息

rbd showmapped

\#查看k8s-pool/foo的详细信息

rbd info k8s-pool/foo

![](media/a15b15c9b2eeaef65bf403a8cd39493b.png)

1.  mount到本地使用

    1.  mount挂载ceph

\#查看已经映射的Block Device信息

rbd showmapped

\#ceph Client的User Space挂载(Mount)该RBD设备 /dev/rbd0到本地目录/mnt

\#把rbd0挂载到本地目录

mount /dev/rbd0 /mnt

\#查询

mount \|grep /dev/rbd0

![](media/c296a292b9b0926bc6aa37cac2288402.png)

### app使用挂载路径

nginx使用ceph挂载目录/mnt进行持久化存储。

cat \> nginx.yaml \<\<EOF

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

name: nginx

\#namespace: kube-system

spec:

replicas: 1

template:

metadata:

labels:

app: nginx

spec:

nodeSelector:

web: nginx

containers:

\- name: nginx

image: nginx:alpine

imagePullPolicy: IfNotPresent

ports:

\- containerPort: 80

volumeMounts:

\- name: httpd-storage

mountPath: /usr/share/nginx/html

volumes:

\- name: httpd-storage

hostPath:

path: /mnt

\---

apiVersion: v1

kind: Service

metadata:

name: nginx

labels:

app: nginx

spec:

type: NodePort

ports:

\- port: 80

nodePort: 32001

protocol: TCP

selector:

app: nginx

EOF

\#创建app

kubectl create -f nginx.yaml

\#测试app

curl http://192.168.120.176:32001/

![](media/56bc69858c9b8be287ebde1079f0a357.png)

1.  pv/pvc方式使用

    1.  创建pv/pvc

grep key /etc/ceph/ceph.client.admin.keyring \|awk '{printf "%s", \$NF}'\|base64

QVFCMjROWmEyRDRYSFJBQTlLNjhmeXJydTkvL2h5dzNEcDBwZWc9PQ==

cat \> ceph-secret.yaml \<\<EOF

apiVersion: v1

kind: Secret

metadata:

name: ceph-secret

namespace: default

type: "kubernetes.io/rbd"

data:

key: QVFCZmdTcFRBQUFBQUJBQWNXTmtsMEFtK1ZkTXVYU21nQ0FmMFE9PQ==

EOF

cat \> pv.yaml \<\<EOF

apiVersion: v1

kind: PersistentVolume

metadata:

name: test-pv

namespace: default

spec:

capacity:

storage: 1Gi

accessModes:

\- ReadWriteOnce

rbd:

monitors:

\- '192.168.120.173:6789'

pool: k8s-pool

image: foo

fsType: xfs

readOnly: false

user: admin

secretRef:

name: ceph-secret

persistentVolumeReclaimPolicy: Recycle

EOF

cat \> pvc.yaml \<\<EOF

kind: PersistentVolumeClaim

apiVersion: v1

metadata:

name: test-pvc

namespace: default

spec:

accessModes:

\- ReadWriteOnce

resources:

requests:

storage: 1Gi

EOF

\#创建 pvc/pv

kubectl create -f ceph-secret.yaml

kubectl create –f pv.yaml

kubectl create –f pvc.yaml

![](media/aacbda533db5678d8b2c98a3f7628129.png)

![](media/3fb11f575b60befb926334d12a4673fe.png)

### 创建pod

\# pod中目录/usr/share/nginx/html映射到ceph集群

cat \> nginx.yaml \<\<EOF

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

name: nginx-dm

namespace: default

spec:

replicas: 1

template:

metadata:

labels:

name: nginx

spec:

containers:

\- name: nginx

image: nginx:alpine

imagePullPolicy: IfNotPresent

ports:

\- containerPort: 80

volumeMounts:

\- name: ceph-rbd-volume

mountPath: "/usr/share/nginx/html"

volumes:

\- name: ceph-rbd-volume

persistentVolumeClaim:

claimName: test-pvc

EOF

\#创建pod

kubectl create -f nginx.yaml

\#进入pod操作

![](media/e99585cec0c418153dedd1269dcbe20b.png)

1.  StorageClass方式使用

    1.  创建StorageClass

grep key /etc/ceph/ceph.client.admin.keyring \|awk '{printf "%s", \$NF}'\|base64

QVFCMjROWmEyRDRYSFJBQTlLNjhmeXJydTkvL2h5dzNEcDBwZWc9PQ==

cat \> ceph-secret.yaml \<\<EOF

apiVersion: v1

kind: Secret

metadata:

name: ceph-secret-admin

namespace: kube-system

type: kubernetes.io/rbd

data:

key: QVFCMjROWmEyRDRYSFJBQTlLNjhmeXJydTkvL2h5dzNEcDBwZWc9PQ==

EOF

cat \> rbd-class.yaml \<\<EOF

apiVersion: storage.k8s.io/v1

kind: StorageClass

metadata:

name: fast

namespace: kube-system

provisioner: ceph.com/rbd

parameters:

monitors: 192.168.120.173:6789

adminId: admin

adminSecretName: ceph-secret-admin

adminSecretNamespace: kube-system

pool: k8s-pool

userId: admin

userSecretName: ceph-secret-admin

imageFormat: "2"

imageFeatures: layering

fsType: xfs

EOF

cat \> ceph-pvc.yaml \<\<EOF

kind: PersistentVolumeClaim

apiVersion: v1

metadata:

name: ceph-claim-dynamic

namespace: kube-system

spec:

accessModes:

\- ReadWriteOnce

resources:

requests:

storage: 1Gi

storageClassName: fast

EOF

cat \> rbd-provisioner.yaml \<\<EOF

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

name: rbd-provisioner

namespace: kube-system

spec:

replicas: 1

template:

metadata:

labels:

app: rbd-provisioner

spec:

containers:

\- name: rbd-provisioner

image: "quay.io/external_storage/rbd-provisioner:latest"

serviceAccountName: persistent-volume-binder

EOF

\#创建storageclass，rbd-provisioner.yaml创建成功后pvc状态变为Bound

kubectl create -f ceph-secret.yaml

kubectl create -f rbd-class.yaml

kubectl create -f ceph-pvc.yaml

kubectl create -f rbd-provisioner.yaml

![](media/5387cf2eb00dbe4a9d22cf3a73619307.png)

### 创建deployment

\#pod中目录/usr/share/nginx/html映射到ceph集群

cat \> deployment.yaml \<\<EOF

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

name: nginx-dynamic

namespace: kube-system

spec:

replicas: 1

template:

metadata:

labels:

name: nginx

spec:

containers:

\- name: nginx

image: nginx:alpine

imagePullPolicy: IfNotPresent

ports:

\- containerPort: 80

volumeMounts:

\- name: ceph-rbd-dynamic-volume

mountPath: /usr/share/nginx/html

volumes:

\- name: ceph-rbd-dynamic-volume

persistentVolumeClaim:

claimName: ceph-claim-dynamic

EOF

kubectl create -f deployment.yaml

![](media/c3fcecd2cbce76f8c1e0613216fb5724.png)

1.  附录

    1.  参考

-   基于docker部署ceph以及修改docker image

https://ceph.com/planet/%E5%9F%BA%E4%BA%8Edocker%E9%83%A8%E7%BD%B2ceph%E4%BB%A5%E5%8F%8A%E4%BF%AE%E6%94%B9docker-image/
