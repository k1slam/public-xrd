# What is this Repository

This repository deploys an IOS-XRd network on your Single Node OpenShift (SNO) cluster and preconfigures all XRd routers to create Segment Routing MPLS (SR-MPLS) network, including one XRd instance that used as SR-PCE (Segment Routing Path Computation Element). The IGP and BGP pre-configuration is based on Seamless MPLS (a.k.a Unified MPLS) architecture and uses BGP Labelled Unicast (BGP-LU) as defined in **RFC-3107: Carrying Label Information in BGP-4**

It is intended as a proof of concept for running Cisco XRd based SR-PCE. This network can be used as a learning tool for Seamless MPLS and Segement Routing technolgies without the need of a physical network. All you need is a cheap Dell/HPE/SuperMicro/UCS server along with access to Red Hat OpenShift (Open and Free for Evaluation) and Cisco XRd softwares.

The following  topology comprising 7 XRd instances is created by succesfully executing this reposity. Notice the use of **Linux Bridge** for point-to-point link between routers, which are mapped to IOS-XR interfaces.connectivity each router. 

> This network is fully contained within a Singe Node Openshift (SNO) cluster. It does not implement, nor require, any external connectivity

![image](https://github.com/user-attachments/assets/faf8fddc-c63f-4ca8-a358-8b9de223c902)

Upon successfull execution of the repo, the included configmaps will pre-configure the XRd instances to create the following IGP and BGP Topology, along with the PCE-PCC peering session using PCEP as shown in the topology:

![image](https://github.com/user-attachments/assets/ecbec36e-15a5-4201-a77e-7b98958b2b09)

PCE: Path Compuation Element
PCC: Path Computation Client
PCEP: Path Computation Element Protocol


## Pre-Requisites:
To use this repository, you will need the following: 

1. A Single Node OpenShift (SNO) Cluster with approximately 500 CPU millicores 2GB Memory for each XRd workloads. This is an estimate and acctual needs could be lower or higher. 
2. A bastion host to access the SNO, store the yamls file and excute `oc` commands. I use a Raspberry Pi, you can use anything - Windows command prompt, Mac terminal, or any linux host
3. SNO tuned for XRd workloads as outlined in our previous [Blog](https://cloudify.network/nw_xrd_cp.html "Cloudify Dot Network: Deploying Cisco XRd on Red Hat OpenShift") and on Syed Hassan's [Public XRd Repo](https://github.com/git-shassan/xrd-public "Syed Hassan's XRd Repo")
4. Access to Cisco XRd Software on a registry accessible from your SNO


## How to Use this repository: 
To successfully execute this repo and create your own XRd based BGP topology, you will need to perform these 4 steps: 

1. Download/copy the files to your bastion host
2. Make required changes:
    * Change the NAD to reflect your own server interface instead of `eno4`, if applicable
    * Update the `statefulsets` to point to your XRd software in your registry
3. Apply the manifests for NAD, ConfigMaps, and StatefulSets for all routers
4. Verify XRd pod instantiation, login to the routers, and verify BGP configuration

### Step 1 of 4: Copy the Files:
Once all the prerequisites are met, you can clone the repo or download the yaml files to your local terminal. Make sure this local host has access to the cluster using `oc` cli

```
admin@mipi4:~/git/XRd/bgp-xrd $ ls -l
-rw-r--r-- 1 admin admin  578 Sep 11  2024 bgp-nad.yaml
-rw-r--r-- 1 admin admin  949 Sep 12  2024 r1-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r1-statefulset.yaml
-rw-r--r-- 1 admin admin  941 Sep 11  2024 r2-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r2-statefulset.yaml
-rw-r--r-- 1 admin admin  943 Sep 12  2024 r3-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r3-statefulset.yaml
-rw-r--r-- 1 admin admin  936 Sep 12  2024 r4-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r4-statefulset.yaml
-rw-r--r-- 1 admin admin  941 Sep 12  2024 r5-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r5-statefulset.yaml
-rw-r--r-- 1 admin admin  941 Sep 12  2024 r6-config.yaml
-rw-r--r-- 1 admin admin 1734 Sep 11  2024 r6-statefulset.yaml
-rw-r--r-- 1 admin admin    5 Apr 19 03:57 Readme.md
-rw-r--r-- 1 admin admin 1316 Sep 12  2024 rr1-config.yaml
-rw-r--r-- 1 admin admin 1740 Sep 11  2024 rr1-statefulset.yaml
-rw-r--r-- 1 admin admin 1316 Sep 29  2024 rr2-config.yaml
-rw-r--r-- 1 admin admin 1740 Sep 11  2024 rr2-statefulset.yaml
```


###  Step 2 of 4:  Make Required Changes: 
There are two changes that needs to be makde to the files: 
1. Change the NAD to reflect your setup
2. Point the `statefulsets` to your registry

Here are the steps to make these changes. 

#### Step 2a: Changing the Interface on NAD

The file `bgp-nad.yaml` contains the `NetworkAttachmentDefinition` which defines the Linux Bridge used for connectivity across the XRd instances. The Linux Bridge `br-1` uses the interface `eno4` for externer connectivity. This is the inerface our XRd instances will use to connect to external networks, if required. 

Here is the snippet of the NetworkAttachementDefinition used in this topology. 

```
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: br-1
  namespace: xrd
spec:
  config: '{ "cniVersion": "0.3.1", "name": "br-1", "type": "macvlan", "mode": "bridge", "master": "eno4", "ipam": { }}'
```

First, you might want to figure out the interface name of the interface you want to use for external connectivity. You can use `oc debug` to interact with the host libraries and identify how the interfaces are seen by the system. Here it shows how: 

```
admin@mipi4:~ $ oc get node
NAME         STATUS   ROLES                         AGE    VERSION
kislam-sno   Ready    control-plane,master,worker   257d   v1.29.7+4510e9c

admin@mipi4:~ $ oc debug node/kislam-sno
Temporary namespace openshift-debug-hcwh2 is created for debugging node...
Starting pod/kislam-sno-debug-lwmh4 ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.86.35
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1#
sh-5.1# ip -br a
lo               UNKNOWN        
eno3             UP
eno4             UP             x.x.x.x/24 << This is my external facing interface. 
eno1             DOWN
eno2             DOWN
```

Update the `bgp-nad.yaml` file with the interface name from your output. In the example above, `eno4` is used. In your scenarion, pick the interface yhat you've connected to your upstream router


#### Step 2b: Updating the Statefulset
The `statfulset.yaml` are the ones that will instantiate the XRd pod. There is a separate stateful set for each router in the topology. However, these files ***do NOT*** contain the location of the XRd image that will be used. This is because XRd software is Cisco property, and you should have the entitlements to use it. Once you've downloaded the images and put in your image registry, you **must** edit the statefulset to reflect the location of the image. 

Look for the text `REPLACE_ME` in the YAML files and replace it with the XRd image location and version in your image repository

```
admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ cat r1-statefulset.yaml | grep REPLACE_ME
      - image: "REPLACE_ME"
```

Once all the statefulsets are edited, proceed to the next step. 


### Step 3 of 4: Apply the Required Manifests
**The SNO installation must be tuned as specified at https://cloudify.network/nw_xrd_cp.html for this step to work**
You will need to do the following (in sequence) to deploy the XRd topology: 

1. Create the namespace (we use name `xrd`) 
2. Create a serviceaccount (we use `xrd-sa`)
3. Give the account appropriate privileges
4. Create Network Attachment Definition (NAD)
5. Create configmap for each router using individual configmap files
6. Instantiate each XRd instance using individual statefulset files


Here is the sample output of perfroming all above tasks:

```
admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f ns.yaml
namespace/xrd created

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f sa.yaml
serviceaccount/xrd-sa created

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f role.yaml
clusterrolebinding.rbac.authorization.k8s.io/system:openshift:scc:privileged configured

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f bgp-nad.yaml
networkattachmentdefinition.k8s.cni.cncf.io/br-1 created

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f r1-config.yaml
configmap/r1-config created

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc apply -f r1-statefulset.yaml
statefulset.apps/r1 created
```

**You will need to apply configmaps and statefulset for all the routers in the topology individually. 
Alternatively, you can create a kustomize file and apply all these manifests at once**




### Step 4 of 4: Verfiy XRd Pods and Connect to Router(s)
Once you've applied all the manifests, provided your XRd image repository is accessible to the SNO, all XRd pods should be instantiated as shown below: 

```
admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc project xrd
Now using project "xrd" on server "https://api.kislam-sno.example.com:6443".

admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc get pod
NAME    READY   STATUS    RESTARTS   AGE
r1-0    1/1     Running   0          22m
r2-0    1/1     Running   0          49s
r3-0    1/1     Running   0          49s
r4-0    1/1     Running   0          48s
r5-0    1/1     Running   0          8s
r6-0    1/1     Running   0          8s
rr1-0   1/1     Running   0          7s
rr2-0   1/1     Running   0          7s
```

You can log in to any XRd application and play around with your newly created BGP network. Here is how to log into the routers: 

```
admin@mipi4:~/git/public-xrd/xrd-bgp-basic $ oc exec -it r1-0 -- xr

User Access Verification

Username: redhat
Password: redhat123


RP/0/RP0/CPU0:XRd1#
RP/0/RP0/CPU0:XRd1#sh ip bgp summ
Mon May 26 05:14:28.257 UTC
BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 18
BGP main routing table version 18
BGP NSR Initial initsync version 5 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker              18         18         18         18          18           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.86.251    0   100      20      12       18    0    0 00:09:07          8
192.168.86.252    0   100      20      12       18    0    0 00:09:03          8

RP/0/RP0/CPU0:XRd1#
```

**Note**: 
All the routers in this topology have external connectivity - courtesy of the interface specified in the `NetworkAttahcmentDefinition`. This can be verified by checking the routing table and trying an external ping:


```
RP/0/RP0/CPU0:XRd1#sh ip route static
S*   0.0.0.0/0 [1/0] via 192.168.86.1, 00:34:52

RP/0/RP0/CPU0:XRd1#sh ip int br
Interface                      IP-Address      Status          Protocol Vrf-Name
Loopback0                      1.1.1.1         Up              Up       default
GigabitEthernet0/0/0/0         192.168.86.241  Up              Up       default

RP/0/RP0/CPU0:XRd1#ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8 timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 9/9/11 ms
RP/0/RP0/CPU0:XRd1#
```

Upstream router (the default gateway in this setup) is configured to use 192.168.86.1/24. The IP addressing across the network will have to be changed if your router IP is differnet. However, the external connectivity have no bearing on the rest of the BGP network, so you can play around with the network topology if you do NOT require external connectivity. 

In other blogs and repo's we rely on connectivity between two SNO's, in which case using appropriate IP addressing scheme will be required. 
