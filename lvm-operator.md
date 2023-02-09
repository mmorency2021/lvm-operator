![](media/image1.png){width="6.5in" height="3.7222222222222223in"}

![](media/image2.png){width="6.5in" height="3.7222222222222223in"}

Server disk configuration

\[root\@master-0 \~\]\# lsblk

NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT

sdb 8:16 0 446.6G 0 disk

├─sdb1 8:17 0 1M 0 part

├─sdb2 8:18 0 127M 0 part

├─sdb3 8:19 0 384M 0 part /boot

└─sdb4 8:20 0 446.1G 0 part /sysroot

sdc 8:32 0 446.6G 0 disk

export KUBECONFIG=localhost.kubeconfig

\[root\@master-0 node-kubeconfigs\]\# oc get nodes

NAME STATUS ROLES AGE VERSION

master-0.sno-ocp12.hubcluster-1.lab.eng.cert.redhat.com Ready
control-plane,master,worker 5d16h v1.25.4+77bec7a

1- First step is to install ODF LVM Operator

We will do it from Operator Hub

![](media/image4.png){width="6.5in" height="3.25in"}

![](media/image3.png){width="6.5in" height="3.25in"}

2- validate that operator pods are running fine

oc project openshift-storage

Now using project \"openshift-storage\" on server
\"https://localhost:6443\".

\[root\@master-0 node-kubeconfigs\]\# oc get pods

NAME READY STATUS RESTARTS AGE

lvm-operator-controller-manager-87b6dfcc6-nq9s6 3/3 Running 0 2m41s

3- next step is to create a LVM cluster

We will use below Custom Ressources (CR) samples or we can use GUI

\[root\@master-0 node-kubeconfigs\]\# cat lvm-cluster.yaml

apiVersion: lvm.topolvm.io/v1alpha1

kind: LVMCluster

metadata:

name: odf-lvmcluster

namespace: openshift-storage

spec:

storage:

deviceClasses:

\- thinPoolConfig:

sizePercent: 90

name: thin-pool-1

overprovisionRatio: 10

name: vg1

\# oc apply -f lvm-cluster.yaml

Wait for al the pods to come up , it will take a while

\[root\@master-0 node-kubeconfigs\]\# oc get pods -w

NAME READY STATUS RESTARTS AGE

lvm-operator-controller-manager-87b6dfcc6-nq9s6 3/3 Running 0 9m55s

topolvm-controller-7d58cb79f8-8j6lh 5/5 Running 0 64s

topolvm-node-kxlgb 0/4 Init:0/1 0 64s

vg-manager-7l6r9 1/1 Running 0 64s

oc get pods -n openshift-storage

NAME READY STATUS RESTARTS AGE

lvm-operator-controller-manager-87b6dfcc6-nq9s6 3/3 Running 0 11m

topolvm-controller-7d58cb79f8-8j6lh 5/5 Running 0 2m30s

topolvm-node-kxlgb 4/4 Running 0 2m30s

vg-manager-7l6r9 1/1 Running 0 2m30s

You should see a new Sc ( storageclass created and waiting for first
consumer)

\[root\@master-0 node-kubeconfigs\]\# oc get sc

NAME PROVISIONER RECLAIMPOLICY VOLUMEBINDINGMODE ALLOWVOLUMEEXPANSION
AGE

odf-lvm-vg1 (default) topolvm.cybozu.com Delete WaitForFirstConsumer
true 3m9s

a Volume Group is created containing all the available Physical Volumes
(disks).

root\@master-0 node-kubeconfigs\]\# vgs

VG #PV #LV #SN Attr VSize VFree

vg1 1 1 0 wz\--n- 446.62g 44.66g

\[root\@master-0 node-kubeconfigs\]\# pvs

PV VG Fmt Attr PSize PFree

/dev/sdc vg1 lvm2 a\-- 446.62g 44.66g

If we look back to the server\'s storage configuration:

\[root\@master-0 node-kubeconfigs\]\# lsblk

NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT

sdb 8:16 0 446.6G 0 disk

├─sdb1 8:17 0 1M 0 part

├─sdb2 8:18 0 127M 0 part

├─sdb3 8:19 0 384M 0 part /boot

└─sdb4 8:20 0 446.1G 0 part /sysroot

sdc 8:32 0 446.6G 0 disk

├─vg1-thin\--pool\--1_tmeta 253:0 0 52M 0 lvm

│ └─vg1-thin\--pool\--1 253:2 0 401.9G 0 lvm

└─vg1-thin\--pool\--1_tdata 253:1 0 401.9G 0 lvm

└─vg1-thin\--pool\--1 253:2 0 401.9G 0 lvm

sr0 11:0 1 104.1M 0 rom

It is taking sdc but not sdb because is occupied by the OS.

\$\> lsblk /dev/nvme1n1 -o NAME,ROTA,TYPE,SIZE,FSTYPE

### **Testing the storage**

We can now create a POD that will use the sc

First let us create a PVC

\[root\@master-0 node-kubeconfigs\]\# cat pvc.yaml

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

name: lvmpvc

labels:

type: local

spec:

storageClassName: odf-lvm-vg1

resources:

requests:

storage: 5Gi

accessModes:

\- ReadWriteOnce

volumeMode: Filesystem

Oc apply -f pvc.yaml

\[root\@master-0 node-kubeconfigs\]\# oc get pvc

NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE

lvmpvc Pending odf-lvm-vg1 4s

The pvc is waiting for a pod to consume it

Now using below manifest , let us create a pod

\[root\@master-0 node-kubeconfigs\]\# cat pod-test.yaml

apiVersion: v1

kind: Pod

metadata:

name: lvmpod

spec:

volumes:

\- name: storage

persistentVolumeClaim:

claimName: lvmpvc

containers:

\- name: container

image: public.ecr.aws/docker/library/nginx:latest

ports:

\- containerPort: 80

name: \"http-server\"

volumeMounts:

\- mountPath: \"/usr/share/nginx/html\"

name: storage

oc apply -f pod-test.yaml

\[root\@master-0 node-kubeconfigs\]\# oc get pods

NAME READY STATUS RESTARTS AGE

lvmpod 1/1 Running 0 6s

\[root\@master-0 node-kubeconfigs\]\# oc get pods

NAME READY STATUS RESTARTS AGE

lvmpod 1/1 Running 0 6s

\[root\@master-0 node-kubeconfigs\]\# oc get pvc

NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE

lvmpvc Bound pvc-f169c633-ebe5-4666-8916-400477f77791 5Gi RWO
odf-lvm-vg1 75s

\[root\@master-0 node-kubeconfigs\]\#

\[root\@master-0 node-kubeconfigs\]\# oc get pv

NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS
REASON AGE

pvc-f169c633-ebe5-4666-8916-400477f77791 5Gi RWO Delete Bound
default/lvmpvc odf-lvm-vg1 29s
