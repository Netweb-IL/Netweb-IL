# Pod Extension - St Albans cth03

This paper covers the process of an FCE deployed site expansion with pod2 physical nodes.

Extension process high-level overview:
- rack and cable / configure new hardware
- prerequisites validation
- enlist new nodes using maas
- update node configuration in maas (disk layout, networks, filesystems)
- magpie network validation
- extend controller nodes and relocate services
- add new compute nodes
- add new ceph storage nodes
- post configuration steps (sriov, sysconfig, etc.)
- validation and testing

# Hardware deployment

BT Engineers will rack and cable nodes based on DDD and configure switches.
The DDD must be updated to include host names, positions (pod and zone),
bmc credentials.

Configure BIOS to allow DHCP boot on OAM interface.


# Prerequisites validation

## CPU Model and CPU Core layout

Make sure the cpu model and core layout are matching between the original pods and new pod. If the layout is different, the nova-compute-kvm applications should be split out to a new one with the settings reflecting the new cpu characteristics.

## Network ranges and available IP addresses

It is important to check that the network spaces has enough free ip address for magpie testing and openstack bundle deployment. The magpie test requires 4 ips from oam network per node.

## IPMI access

Make sure that all new pod instances ipmi interface is reachable from infra nodes.

## Model status

The juju status must be all-green and free from any error or warning messages.

## Check for uncommitted changes

Make sure no uncommitted changes are present on the infra node's local git
repository:

```
$ cd ~/deployment/stalbans-cth03-canonical
$ git status
On branch canonical
Your branch is up to date with 'origin/canonical'.

nothing to commit, working tree clean
```

Notice: the local git repository should be present on the jumphost instead of 
the infra nodes on older deployments.


# Enlist new nodes using maas

## Update nodes configuration file

The bmc credentials, tags, zone information of new pod nodes must be appended
to the nodes.yaml file. Make sure to include tags for class, pod and az. The
node availability zone must be present as a tag definition and as a zone
definition, double check that those values are matching the DDD.

If this deployment is an expansion of a pod1 only site, as a first step,
append the pod1 and zone tag - node class tag if missing - to each existing
nodes:

```
  stanc01node002:
    bmc_user: include-rel://secrets/cimc-user
    bmc_password: include-rel://secrets/cimc-password
    power_type: ipmi
    bmc_address: 10.15.145.5
    bmc_power_boot_type: efi
++  tags: ['openstack','pod1','az1']
    zone: az1
```

Validate zone and tag az assignment, where the tags and zone must match:

```

$ cat config/nodes.yaml | egrep -A1 "pod1','az1" # check az1 nodes
$ cat config/nodes.yaml | egrep -A1 "pod1','az2" # check az2 nodes
$ cat config/nodes.yaml | egrep -A1 "pod1','az3" # check az3 nodes
```

Expected output:
```
...
  tags: ['ovs','pod1','az3']
  zone: az3
--
  tags: ['ovs','pod1','az3']
  zone: az3
```


Class values:
- 'openstack': openstack controller node
- 'contrail': contrail controller node
- 'storage': ceph storage node
- 'sriov': sriov compute node
- 'dpdk': dpdk compute node
- 'ovs': ovs compute node

Pod values:
- 'pod1'
- 'pod2'
- 'pod3'

Availability zone values:
- 'az1'
- 'az2'
- 'az3'


Add pod2 and pod3 controller, storage and compute nodes:

```yaml
#storage nodes(expansion-pod3)
ltnnc01node091:
  bmc_user: include-rel://secrets/cimc-user
  bmc_password: include-rel://secrets/cimc-password
  power_type: ipmi
  bmc_address: 10.64.1.94
  bmc_power_boot_type: efi
  zone: az1
  tags: ['storage','pod3','az1']
...
#compute servers -pod3
ltnnc01node085:
  bmc_user: include-rel://secrets/cimc-user
  bmc_password: include-rel://secrets/cimc-password
  power_type: ipmi
  bmc_address: 10.64.1.88
  bmc_power_boot_type: efi
  zone: az1
  tags: ['sriov','pod3','az1']
...
```

Validate zone and hostname allocations, compare with DDD:

Check pod2 nodes in az1, then repeat for az2 and az3:

```
$ cat config/nodes.yaml | grep -B7 "pod2','az1" | grep stanc01node | sort -u
```

Check pod2 bmc ip addresses:

```
$ HOSTNAMES=$(cat config/nodes.yaml | grep -B7 "pod2'" | grep stanc01node | sort -u |cut -c 1-14)
$ for i in $HOSTNAMES
do
  echo $i $(cat config/nodes.yaml | grep -A7 "$i" | grep bmc_address)
done
```

Validate zone assignments:

```
$ cat config/nodes.yaml | egrep -A1 "pod2','az1" # check az1 nodes
$ cat config/nodes.yaml | egrep -A1 "pod2','az2" # check az2 nodes
$ cat config/nodes.yaml | egrep -A1 "pod2','az3" # check az3 nodes
```

##

juju status |egrep “error|block|unknown|down|inactive|mainte|seal|hook|waiting|reboot required”

## Validate CIMC access

```
HOSTNAMES=$(cat config/nodes.yaml | egrep -B7 "pod2|pod3'" | grep stanc01node | sort -u)
HOSTNAMES=$(cat config/nodes.yaml | egrep -B7 "pod2'" | grep stanc01node | sort -u)
CIMC_PASSWORD=$(cat secrets/cimc-password)
for i in $HOSTNAMES
do
  bmc_address=$(cat config/nodes.yaml | grep -A7 "$i" | grep bmc_address | awk '{ print $2}')
  echo -n "$i "
  ipmipower -D LAN_2_0 -h $bmc_address -u admin -p $CIMC_PASSWORD --stat
done
unset $CIMC_PASSWORD
```

## Power off new nodes

```
HOSTNAMES=$(cat config/nodes.yaml | egrep -B7 "pod2'" | grep stanc01node | sort -u)
CIMC_PASSWORD=$(cat secrets/cimc-password)
for i in $HOSTNAMES
do
  bmc_address=$(cat config/nodes.yaml | grep -A7 "$i" | grep bmc_address | awk '{ print $2}')
  echo -n "$i "
  ipmipower -D LAN_2_0 -h $bmc_address -u admin -p $CIMC_PASSWORD --off
done
unset $CIMC_PASSWORD
```

juju status |egrep “error|block|unknown|down|inactive|mainte|seal|hook|waiting|reboot required”

ipmitool -H BMC-IP-ADDRESS -I lanplus -U USER -P PASSWORD chassis bootdev pxe;

## Enlist new nodes

Enlist the pod2 nodes using FCE:

```
$ fce --debug build --layer maas --steps maas:enlist_nodes --force-rebuild
```

## Configure enlisted nodes

### Rebuild buckets.yaml

This step is collecting the network, storage layout from the Maas machine
inventory and generating the generated/buckets.yaml file.

```
$ fce --debug build --layer maas --steps maas:generate_buckets --force-rebuild
```

### Prepare bucketsconfig.yaml

`important:` add new nodes, update hostname - static ip assignments.

Append the new nodes to the allocations section:

```yaml
---
allocations:

  block-storage-nodes:
    machines:
      - ltnnc01node091.502.nci.bt.com
      - ltnnc01node092.502.nci.bt.com
      - ltnnc01node093.502.nci.bt.com
      ...

  compute-nodes-group1:
    machines:
      - ltnnc01node085.502.nci.bt.com
      - ltnnc01node086.502.nci.bt.com
      - ltnnc01node087.502.nci.bt.com
      ...

```


### Apply node configuration

This step is configuring the network and storage for maas enlisted nodes. 

```
$ fce --debug build --layer maas --steps maas:configure_nodes --force-rebuild
```


### Magpie validation

Prepare a separate magpie file including the new pod`s nodes:

magpie-bundle.yaml.pod2:

```yaml
---                                                             
series: "bionic"
```
Before Build and deploy magpie, check proxy, 

$ env |grep proxy

If proxy not set, set proxy-

sudo vim /etc/environments
source /etc/environments


Build and deploy the new magpie model:

```
$ juju add-model magpie --config ./config/juju-model-default.yaml
$ juju deploy -m magpie ./config/magpie-bundle.yaml.pod2 --dry-run
$ juju deploy -m magpie ./config/magpie-bundle.yaml.pod2
$ juju-wait -v
```


# Controller node expansion

Additional 3 controller nodes will be added as part of this change, the total number of Openstack Controllers will be 6.

Controller machines:
- 100 (pod1, az1)
- 101 (pod1, az2)
- 102 (pod1, az3)
- 110 (pod2, az1) - new
- 111 (pod2, az2) - new
- 112 (pod2, az3) - new

Expected service placement:

| Application name              | 100 | 101 | 102 | 110 | 111 | 112 |
|-------------------------------|-----|-----|-----|-----|-----|-----|
| etcd                          |     |     |     |  X  |  X  |  X  |
| ceph-radosgw                  |  X  |  X  |  X  |  X  |  X  |  X  |
| aodh                          |  X  |  X  |  X  |     |     |     |
| gnocchi                       |     |     |     |  X  |  X  |  X  |
| cinder                        |  X  |  X  |  X  |     |     |     |
| glance                        |  X  |  X  |  X  |     |     |     |
| keystone                      |  X  |  X  |  X  |     |     |     |
| mysql                         |  X  |  X  |  X  |     |     |     |
| neutron-api                   |     |     |     |  X  |  X  |  X  |
| nova-cloud-controller         |  X  |  X  |  X  |     |     |     |
| openstack-dashboard           |     |     |     |  X  |  X  |  X  |
| rabbitmq-server               |     |     |     |  X  |  X  |  X  |
| memcached                     |     |     |     |  X  |  X  |  X  |
| ceilometer                    |     |     |     |  X  |  X  |  X  |
| openstack-service-checks      |  X  |     |     |     |     |     |
| prometheus-openstack-exporter |     |     |  X  |     |     |     |
| prometheus-ceph-exporter      |  X  |     |     |     |     |     |
| heat                          |  X  |  X  |  X  |     |     |     |

# Add pod2 controller nodes

Add controller machines:

```
$ juju add-machine -m openstack --series bionic --constraints tags=foundation-nodes,controller-nodes,pod2,az1
$ juju add-machine -m openstack --series bionic --constraints tags=foundation-nodes,controller-nodes,pod2,az2
$ juju add-machine -m openstack --series bionic --constraints tags=foundation-nodes,controller-nodes,pod2,az3

```

Wait for machines to be settled:
```
$ juju machines -m openstack | tail -n 4
65         started  10.15.144.243  stanc01node043          bionic  az1  Deployed
66         started  10.15.144.244  stanc01node045          bionic  az2  Deployed
67         started  10.15.144.245  stanc01node047          bionic  az3  Deployed
```

Set variables for further expansion procedures:

```
$ . scripts/expansion/get-pod2-controller-nodes.sh
$ echo $CTRL_POD2_NODES
```


## Extend controller server units

```
$ for unit_id in $CTRL_POD2_NODES
do
  juju add-unit -m openstack controller-server -n 1 --to $unit_id
done
$ juju-wait -m openstack -v
```



Add bundle machine definition for controller nodes:


```
machines:
...
  # Additional controller nodes in Pod2
  "110":
    constraints: tags=foundation-nodes,controller-nodes,pod2,az1
  "111":
    constraints: tags=foundation-nodes,controller-nodes,pod2,az2
  "112":
    constraints: tags=foundation-nodes,controller-nodes,pod2,az3
```

Add new units (machine id: 110, 111, 112) to server-controller application and
increase unit number to 6.


```
applications:
  #Just to depoy ubuntu charm for nagios monitoring of baremetal nodes
  controller-server:
    charm: cs:ubuntu
    num_units: 6
    to:
      - 102
      - 104
      - 106
      - 110
      - 111
      - 112
```

Appy changes and deploy new controller nodes:

```
$ juju deploy ./config/bundle.yaml --overlay config/overlays/ldap.yaml --overlay config/overlays/hostnames.yaml --overlay config/overlays/ssl.yaml --overlay config/overlays/ovs.yaml --overlay config/overlays/openstack_versioned_overlay.yaml
```


# Relocate openstack dashboard service

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id openstack-dashboard
done
```

Check service quorum and members:

```
$ juju ssh openstack-dashboard/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Thu Jun 17 21:36:35 2021
Quorum provider:  corosync_votequorum
Nodes:            6
Node ID:          1004
Ring ID:          1000/548
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   6
Highest expected: 6
Total votes:      6
Quorum:           4
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1000          1 10.15.144.206
      1001          1 10.15.144.222
      1002          1 10.15.144.231
      1004          1 10.15.144.246 (local)
      1005          1 10.15.144.247
      1003          1 10.15.144.248
Connection to 10.15.147.90 closed.
```

Remove non-leader units

```
$ juju remove-unit openstack-dashboard/1
$ juju remove-unit openstack-dashboard/2
$ juju-wait -v
```

Set quorum to 3, so the leader unit can be safely removed. 

```
juju ssh openstack-dashboard/3 'sudo corosync-quorumtool -e 3'

juju ssh openstack-dashboard/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Fri Jun 18 07:24:17 2021
Quorum provider:  corosync_votequorum
Nodes:            4
Node ID:          1004
Ring ID:          1001/564
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   4
Highest expected: 4
Total votes:      4
Quorum:           3
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1001          1 10.15.144.222
      1004          1 10.15.144.246 (local)
      1005          1 10.15.144.247
      1003          1 10.15.144.248
Connection to 10.15.144.246 closed.
```

Remove the leader unit:

```
$ juju remove-unit openstack-dashboard/0
$ juju-wait -v
```

Check the quorum status again:

```
$ juju ssh openstack-dashboard/3 'sudo corosync-quorumtool'
$ juju ssh openstack-dashboard/3 'sudo crm_mon -Arf1'
```

clean up pacemaker resource:

```
$ juju ssh openstack-dashboard/3 'sudo crm resource cleanup res_horizon_haproxy'
```

 

remove original nodes with offline status from the cluster:

```
$ juju ssh openstack-dashboard/3 'sudo crm_node -R juju-a2e49d-18-lxd-12 --force'
$ juju ssh openstack-dashboard/3 'sudo crm_node -R juju-a2e49d-19-lxd-11 --force'
$ juju ssh openstack-dashboard/3 'sudo crm_node -R juju-a2e49d-20-lxd-11 --force'
```

Remove the nodes from /etc/corosync.conf:

```
$ juju run -a hacluster-horizon 'hooks/config-changed'
```

Validate quroum and voting values (votes must be 3 and quorum is 2):

```
$ juju ssh openstack-dashboard/3 'sudo corosync-quorumtool' | egrep "Expected|Quorum:"
Expected votes:   3
Quorum:           2
```

Add missing db access ip address:

```
$ juju remove-relation mysql:shared-db openstack-dashboard:shared-db
$ juju add-relation mysql:shared-db openstack-dashboard:shared-db
```

bundle.yaml changes:

```

```


# Relocate gnocchi service

Stretch out services to new controller nodes (3->6) / 37min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id gnocchi
done
$ juju-wait -v
```


Remove non-leader units / 14min:

```
$ juju status gnocchi | grep ^gnocchi/
gnocchi/0*                     active    idle   18/lxd/7  10.15.144.213   8041/tcp       Unit is ready
gnocchi/1                      active    idle   19/lxd/6  10.15.144.230   8041/tcp       Unit is ready
gnocchi/2                      active    idle   20/lxd/6  10.15.144.207   8041/tcp       Unit is ready
gnocchi/3                      active    idle   65/lxd/1  10.15.144.206   8041/tcp       Unit is ready
gnocchi/4                      active    idle   66/lxd/1  10.15.144.222   8041/tcp       Unit is ready
gnocchi/5                      active    idle   67/lxd/1  10.15.144.231   8041/tcp       Unit is ready
```

```
$ juju remove-unit gnocchi/1
$ juju remove-unit gnocchi/2
$ juju-wait -v
```

Set quorum to 3, so the leader unit can be safely removed. 

```
$ juju ssh gnocchi/3 'sudo corosync-quorumtool -e 3'
$ juju ssh gnocchi/3 'sudo corosync-quorumtool'
$ juju ssh gnocchi/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Fri Jun 18 17:28:48 2021
Quorum provider:  corosync_votequorum
Nodes:            4
Node ID:          1005
Ring ID:          1005/904
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   4
Highest expected: 4
Total votes:      4
Quorum:           3  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1005          1 10.15.144.206 (local)
      1001          1 10.15.144.213
      1003          1 10.15.144.222
      1004          1 10.15.144.231
Connection to 10.15.147.138 closed.
```

Remove leader unit / 15min:

```
$ juju remove-unit gnocchi/0
$ juju-wait -v
```

Check the cluster status:
```
$ juju ssh gnocchi/3 'sudo crm_mon -Arf1'
Stack: corosync
Current DC: juju-a2e49d-65-lxd-1 (version 1.1.18-2b07d5c5a9) - partition with quorum
Last updated: Fri Jun 18 17:47:38 2021
Last change: Fri Jun 18 17:33:14 2021 by hacluster via crmd on juju-a2e49d-65-lxd-1

6 nodes configured
8 resources configured

Online: [ juju-a2e49d-65-lxd-1 juju-a2e49d-66-lxd-1 juju-a2e49d-67-lxd-1 ]
OFFLINE: [ juju-a2e49d-18-lxd-7 juju-a2e49d-19-lxd-6 juju-a2e49d-20-lxd-6 ]

Full list of resources:

 Resource Group: grp_gnocchi_vips
     res_gnocchi_1dbca9b_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-1
     res_gnocchi_1fdb8cc_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-1
 Clone Set: cl_res_gnocchi_haproxy [res_gnocchi_haproxy]
     Started: [ juju-a2e49d-65-lxd-1 juju-a2e49d-66-lxd-1 juju-a2e49d-67-lxd-1 ]
     Stopped: [ juju-a2e49d-18-lxd-7 juju-a2e49d-19-lxd-6 juju-a2e49d-20-lxd-6 ]

Node Attributes:
* Node juju-a2e49d-65-lxd-1:
* Node juju-a2e49d-66-lxd-1:
* Node juju-a2e49d-67-lxd-1:

Migration Summary:
* Node juju-a2e49d-67-lxd-1:
* Node juju-a2e49d-66-lxd-1:
* Node juju-a2e49d-65-lxd-1:
Connection to 10.15.144.206 closed.
```

clean up pacemaker resource:

```
$ juju ssh gnocchi/3 'sudo crm resource cleanup res_gnocchi_haproxy'
```

Remove the offline nodes:
```
$ for unit_name in juju-a2e49d-18-lxd-7 juju-a2e49d-19-lxd-6 juju-a2e49d-20-lxd-6
do
  juju ssh gnocchi/5 sudo crm_node -R $unit_name --force
done
```


```
$ juju run -a hacluster-gnocchi 'hooks/config-changed'
```


Check cluster health finally:
```
$ juju ssh gnocchi/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Fri Jun 18 17:51:43 2021
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          1005
Ring ID:          1005/932
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1005          1 10.15.144.206 (local)
      1003          1 10.15.144.222
      1004          1 10.15.144.231
Connection to 10.15.144.206 closed.


$ juju ssh gnocchi/3 'sudo crm_mon -Arf1'
Stack: corosync
Current DC: juju-a2e49d-65-lxd-1 (version 1.1.18-2b07d5c5a9) - partition with quorum
Last updated: Fri Jun 18 17:52:10 2021
Last change: Fri Jun 18 17:49:50 2021 by root via crm_node on juju-a2e49d-67-lxd-1

3 nodes configured
5 resources configured

Online: [ juju-a2e49d-65-lxd-1 juju-a2e49d-66-lxd-1 juju-a2e49d-67-lxd-1 ]

Full list of resources:

 Resource Group: grp_gnocchi_vips
     res_gnocchi_1dbca9b_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-1
     res_gnocchi_1fdb8cc_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-1
 Clone Set: cl_res_gnocchi_haproxy [res_gnocchi_haproxy]
     Started: [ juju-a2e49d-65-lxd-1 juju-a2e49d-66-lxd-1 juju-a2e49d-67-lxd-1 ]

Node Attributes:
* Node juju-a2e49d-65-lxd-1:
* Node juju-a2e49d-66-lxd-1:
* Node juju-a2e49d-67-lxd-1:

Migration Summary:
* Node juju-a2e49d-67-lxd-1:
* Node juju-a2e49d-66-lxd-1:
* Node juju-a2e49d-65-lxd-1:
Connection to 10.15.147.138 closed.
```

# Relocate ceph-radosgw

Stretch out services to new controller nodes (3->6) / 36min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id ceph-radosgw
done
$ juju-wait -v
```

Remove non-leader units / 8min:

```
$ juju status ceph-radosgw | grep ^ceph-radosgw/
ceph-radosgw/0*                active    idle   18/lxd/2  10.15.144.237   80/tcp         Unit is ready
ceph-radosgw/1                 active    idle   19/lxd/2  10.15.144.228   80/tcp         Unit is ready
ceph-radosgw/2                 active    idle   20/lxd/2  10.15.144.225   80/tcp         Unit is ready
ceph-radosgw/3                 active    idle   65/lxd/2  10.15.144.213   80/tcp         Unit is ready
ceph-radosgw/4                 active    idle   66/lxd/2  10.15.144.230   80/tcp         Unit is ready
ceph-radosgw/5                 active    idle   67/lxd/2  10.15.144.207   80/tcp         Unit is ready
```

```
$ juju remove-unit ceph-radosgw/1
$ juju remove-unit ceph-radosgw/2
$ juju-wait -v
```

Set quorum to 3, so the leader unit can be safely removed. 

```
$ juju ssh ceph-radosgw/3 'sudo corosync-quorumtool -e 3'
$ juju ssh ceph-radosgw/3 'sudo corosync-quorumtool'
```

Remove leader unit / 8min:

```
$ juju remove-unit ceph-radosgw/0
$ juju-wait -v
```

Check the cluster status:
```
$ juju ssh ceph-radosgw/3 'sudo crm_mon -Arf1'
```

clean up pacemaker resource:

```
$ juju ssh ceph-radosgw/3 'sudo crm resource cleanup res_cephrg_haproxy'
```

Remove the offline nodes:
```
$ for unit_name in juju-a2e49d-18-lxd-2 juju-a2e49d-19-lxd-2 juju-a2e49d-20-lxd-2
do
  juju ssh ceph-radosgw/5 sudo crm_node -R $unit_name --force
done
```


```
$ juju run -a hacluster-radosgw 'hooks/config-changed'
```

Check cluster health finally:
```
$ juju ssh ceph-radosgw/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Fri Jun 18 20:53:24 2021
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          1003
Ring ID:          1005/3656
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1005          1 10.15.144.207
      1003          1 10.15.144.213 (local)
      1004          1 10.15.144.230
Connection to 10.15.144.213 closed.



$ juju ssh ceph-radosgw/3 'sudo crm_mon -Arf1'
Stack: corosync
Current DC: juju-a2e49d-66-lxd-2 (version 1.1.18-2b07d5c5a9) - partition with quorum
Last updated: Fri Jun 18 20:53:45 2021
Last change: Fri Jun 18 20:50:47 2021 by root via crm_node on juju-a2e49d-67-lxd-2

3 nodes configured
5 resources configured

Online: [ juju-a2e49d-65-lxd-2 juju-a2e49d-66-lxd-2 juju-a2e49d-67-lxd-2 ]

Full list of resources:

 Resource Group: grp_cephrg_vips
     res_cephrg_85c26f1_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-66-lxd-2
     res_cephrg_36c72fc_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-66-lxd-2
 Clone Set: cl_cephrg_haproxy [res_cephrg_haproxy]
     Started: [ juju-a2e49d-65-lxd-2 juju-a2e49d-66-lxd-2 juju-a2e49d-67-lxd-2 ]

Node Attributes:
* Node juju-a2e49d-65-lxd-2:
* Node juju-a2e49d-66-lxd-2:
* Node juju-a2e49d-67-lxd-2:

Migration Summary:
* Node juju-a2e49d-65-lxd-2:
* Node juju-a2e49d-66-lxd-2:
* Node juju-a2e49d-67-lxd-2:
Connection to 10.15.147.87 closed.

$ juju status ceph-radosgw | grep ^ceph-radosgw/
ceph-radosgw/3*                active    idle   65/lxd/2  10.15.144.213   80/tcp         Unit is ready
ceph-radosgw/4                 active    idle   66/lxd/2  10.15.144.230   80/tcp         Unit is ready
ceph-radosgw/5                 active    idle   67/lxd/2  10.15.144.207   80/tcp         Unit is ready
```

# Relocate neutron-api

Notice: the neutron-api service is running in a kvm container, as a legacy of
contrail deployment architectures. (not a requirement for OVS deployments)

Stretch out services to new controller nodes (3->6) / 66min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to kvm:$machine_id neutron-api
done
$ juju-wait -v
```

Remove non-leader units / 24min:

```
$ juju status neutron-api | grep ^neutron-api/
neutron-api/0*                 active    idle   18/kvm/1  10.15.144.239   9696/tcp       Unit is ready
neutron-api/1                  active    idle   19/kvm/1  10.15.144.218   9696/tcp       Unit is ready
neutron-api/2                  active    idle   20/kvm/1  10.15.144.212   9696/tcp       Unit is ready
neutron-api/3                  active    idle   65/lxd/3  10.15.144.225   9696/tcp       Unit is ready
neutron-api/4                  active    idle   66/lxd/3  10.15.144.237   9696/tcp       Unit is ready
neutron-api/5                  active    idle   67/lxd/3  10.15.144.228   9696/tcp       Unit is ready
```

```
$ juju remove-unit neutron-api/1
$ juju remove-unit neutron-api/2
$ juju-wait -v
```

Set quorum to 3, so the leader unit can be safely removed. 

```
$ juju ssh neutron-api/3 'sudo corosync-quorumtool -e 3'
$ juju ssh neutron-api/3 'sudo corosync-quorumtool'

```

Remove leader unit / 24min:

```
$ juju remove-unit neutron-api/0
$ juju-wait -v
```

Check the cluster status:
```
$ juju ssh neutron-api/3 'sudo crm_mon -Arf1'
Stack: corosync
Current DC: juju-a2e49d-65-lxd-3 (version 1.1.18-2b07d5c5a9) - partition with quorum
Last updated: Sun Jun 20 12:24:50 2021
Last change: Sun Jun 20 12:02:40 2021 by hacluster via crmd on juju-a2e49d-65-lxd-3

6 nodes configured
8 resources configured

Online: [ juju-a2e49d-65-lxd-3 juju-a2e49d-66-lxd-3 juju-a2e49d-67-lxd-3 ]
OFFLINE: [ juju-a2e49d-18-kvm-1 juju-a2e49d-19-kvm-1 juju-a2e49d-20-kvm-1 ]

Full list of resources:

 Resource Group: grp_neutron_vips
     res_neutron_104a5d4_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-3
     res_neutron_040a524_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-3
 Clone Set: cl_neutron_haproxy [res_neutron_haproxy]
     Started: [ juju-a2e49d-65-lxd-3 juju-a2e49d-66-lxd-3 juju-a2e49d-67-lxd-3 ]
     Stopped: [ juju-a2e49d-18-kvm-1 juju-a2e49d-19-kvm-1 juju-a2e49d-20-kvm-1 ]

Node Attributes:
* Node juju-a2e49d-65-lxd-3:
* Node juju-a2e49d-66-lxd-3:
* Node juju-a2e49d-67-lxd-3:

Migration Summary:
* Node juju-a2e49d-65-lxd-3:
* Node juju-a2e49d-66-lxd-3:
* Node juju-a2e49d-67-lxd-3:
Connection to 10.15.144.225 closed.
```

clean up pacemaker resource:

```
$ juju ssh neutron-api/3 'sudo crm resource cleanup res_neutron_haproxy'
```

Remove the offline nodes:
```
$ for unit_name in juju-a2e49d-18-kvm-1 juju-a2e49d-19-kvm-1 juju-a2e49d-20-kvm-1
do
  juju ssh neutron-api/5 sudo crm_node -R $unit_name --force
done
```


```
$ juju run -a hacluster-neutron 'hooks/config-changed'
```

Check cluster health finally:
```
$ juju ssh neutron-api/3 'sudo corosync-quorumtool'
Quorum information
------------------
Date:             Sun Jun 20 12:27:49 2021
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          1003
Ring ID:          1003/1108
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
      1003          1 10.15.144.225 (local)
      1004          1 10.15.144.228
      1005          1 10.15.144.237
Connection to 10.15.147.139 closed.



$ juju ssh neutron-api/3 'sudo crm_mon -Arf1'
Stack: corosync
Current DC: juju-a2e49d-65-lxd-3 (version 1.1.18-2b07d5c5a9) - partition with quorum
Last updated: Sun Jun 20 12:28:10 2021
Last change: Sun Jun 20 12:26:20 2021 by root via crm_node on juju-a2e49d-67-lxd-3

3 nodes configured
5 resources configured

Online: [ juju-a2e49d-65-lxd-3 juju-a2e49d-66-lxd-3 juju-a2e49d-67-lxd-3 ]

Full list of resources:

 Resource Group: grp_neutron_vips
     res_neutron_104a5d4_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-3
     res_neutron_040a524_vip	(ocf::heartbeat:IPaddr2):	Started juju-a2e49d-65-lxd-3
 Clone Set: cl_neutron_haproxy [res_neutron_haproxy]
     Started: [ juju-a2e49d-65-lxd-3 juju-a2e49d-66-lxd-3 juju-a2e49d-67-lxd-3 ]

Node Attributes:
* Node juju-a2e49d-65-lxd-3:
* Node juju-a2e49d-66-lxd-3:
* Node juju-a2e49d-67-lxd-3:

Migration Summary:
* Node juju-a2e49d-65-lxd-3:
* Node juju-a2e49d-66-lxd-3:
* Node juju-a2e49d-67-lxd-3:
Connection to 10.15.147.139 closed.

$ juju status neutron-api | grep ^neutron-api/
neutron-api/3                  active    idle   65/lxd/3  10.15.144.225   9696/tcp       Unit is ready
neutron-api/4                  active    idle   66/lxd/3  10.15.144.237   9696/tcp       Unit is ready
neutron-api/5*                 blocked   idle   67/lxd/3  10.15.144.228   9696/tcp       Services not running that should be: haproxy
```

### Issue: lxd instead of kvm

Make sure that always the proper container type is used for applications. Due to Contrail docker requirements BT is using kvm vms for neutron-api and etc. instead of lxd containers. In case of a misconfiguration the lxd containers must be removed and re-created as kvm ones.

```
$ juju remove-unit neutron-api/3
$ juju-wait -v
$ juju add-unit -n 1 --to kvm:65 neutron-api
$ juju-wait -v
$ juju run -a hacluster-neutron 'hooks/config-changed'
```

```
$ juju remove-unit neutron-api/4
$ juju-wait -v
$ juju add-unit -n 1 --to kvm:66 neutron-api
$ juju-wait -v
```

```
$ juju remove-unit neutron-api/5
$ juju-wait -v
$ juju add-unit -n 1 --to kvm:67 neutron-api
$ juju-wait -v
```

```
$ for unit_name in juju-a2e49d-65-lxd-3 juju-a2e49d-66-lxd-3 juju-a2e49d-67-lxd-3
do
  juju ssh neutron-api/6 sudo crm_node -R $unit_name --force
done
```


### Issue: neutron-api/5 Services not running that should be: haproxy

```
juju ssh neutron-api/5 'systemctl status haproxy'● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2021-06-20 12:27:00 UTC; 4min 0s ago
     Docs: man:haproxy(1)
           file:/usr/share/doc/haproxy/configuration.txt.gz
 Main PID: 11828 (haproxy)
    Tasks: 1 (limit: 17203)
   CGroup: /system.slice/haproxy.service
           ├─11828 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           └─11829 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
Connection to 10.15.144.228 closed.
```

Fix: run the config-changed hook on hacluster-neutron units.

```
$ juju run -a hacluster-neutron 'hooks/config-changed'
```


# Relocate etcd

Stretch out services to new controller nodes (3->6) / 21min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id etcd
done
$ juju-wait -v
```

Remove non-leader units / 6min:

```
$ juju status etcd | grep ^etcd/
etcd/0*                  active    idle   18/lxd/5  10.15.147.164   2379/tcp       Healthy with 6 known peers
etcd/1                   active    idle   19/lxd/4  10.15.147.175   2379/tcp       Healthy with 6 known peers
etcd/2                   active    idle   20/lxd/4  10.15.147.144   2379/tcp       Healthy with 6 known peers
etcd/3                   active    idle   65/lxd/4  10.15.147.105   2379/tcp       Healthy with 6 known peers
etcd/4                   active    idle   66/lxd/4  10.15.147.76    2379/tcp       Healthy with 6 known peers
etcd/5                   active    idle   67/lxd/4  10.15.147.86    2379/tcp       Healthy with 6 known peers

```

```
$ juju remove-unit etcd/1
$ juju remove-unit etcd/2
$ juju-wait -v
```

As a second step, the leader should be removed safely, and a new leader
will be elected from the remaining units:

```
$ juju remove-unit etcd/0
```


### Vault service post-configuration steps

By design the vault units are rewriting the vault configuration files with
the new etcd unit ip addresses, however the vault service itself is not going
to be rebooted automatically.

https://bugs.launchpad.net/vault-charm/+bug/1911889

As a post-configuration step, all the vault units must be restarted, then
unsealed again:

```
$ juju run -a vault 'sudo systemctl restart vault'
$ juju-wait -v
```

Wait until the vault service status is showing unseal required status:

```
$ juju status vault | grep ^vault
vault                1.5.4    blocked      3  vault                jujucharms   32  ubuntu  
vault/0                   blocked   idle   15       10.15.147.61    8200/tcp       Unit is sealed
vault/1                   blocked   idle   16       10.15.147.60    8200/tcp       Unit is sealed
vault/2*                  blocked   idle   17       10.15.147.62    8200/tcp       Unit is sealed
```

Unseal all the vault units:

```
$ bash scripts/bt-scripts/vault.sh
```

# Relocate memcached

Stretch out services to new controller nodes (3->6) / 21min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id memcached
done
$ juju-wait -v
```

Remove non-leader units / 6min:

```
$ juju status etcd | grep ^etcd/
memcached                  active      6  memcached         jujucharms   26  ubuntu  
memcached/0*             active    idle   18/lxd/9  10.15.147.108   11211/tcp      Unit is ready
memcached/1              active    idle   19/lxd/8  10.15.147.163   11211/tcp      Unit is ready
memcached/2              active    idle   20/lxd/8  10.15.147.113   11211/tcp      Unit is ready
memcached/3              active    idle   65/lxd/5  10.15.147.130   11211/tcp      Unit is ready and clustered
memcached/4              active    idle   66/lxd/5  10.15.147.103   11211/tcp      Unit is ready and clustered
memcached/5              active    idle   67/lxd/5  10.15.147.97    11211/tcp      Unit is ready and clustered

```

```
$ juju remove-unit memcached/1
$ juju remove-unit memcached/2
$ juju-wait -v
```

As a second step, the leader should be removed safely, and a new leader
will be elected from the remaining units:

```
$ juju remove-unit memcached/0
```

# Relocate ceilometer

Stretch out services to new controller nodes (3->6) 22min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id ceilometer
done
$ juju-wait -v
```

Remove non-leader units / 7min:

```
$ juju status ceilometer | grep ^ceilometer
ceilometer               10.0.1   active      6  ceilometer        jujucharms  268  ubuntu  
ceilometer/0*                  active    idle   18/lxd/1  10.15.144.209                  Unit is ready
ceilometer/1                   active    idle   19/lxd/1  10.15.144.220                  Unit is ready
ceilometer/2                   active    idle   20/lxd/1  10.15.144.205                  Unit is ready
ceilometer/3                   active    idle   65/lxd/6  10.15.144.225                  Unit is ready
ceilometer/4                   active    idle   66/lxd/6  10.15.144.237                  Unit is ready
ceilometer/5                   active    idle   67/lxd/6  10.15.144.228                  Unit is ready
```

```
$ juju remove-unit ceilometer/1
$ juju remove-unit ceilometer/2
$ juju-wait -v
```

As a second step, the leader should be removed safely, and a new leader
will be elected from the remaining units:

```
$ juju remove-unit ceilometer/0
```

# Relocate rabbimtq

Check rabbitmq balance:

```
$ juju run -u rabbitmq-server/leader -- rabbitmqctl list_queues -p openstack pid | sed -e 's/<\([^.]*\).*/\1/' | sort | uniq -c
      1 Listing queues
    676 rabbit@juju-a2e49d-18-lxd-16
    642 rabbit@juju-a2e49d-19-lxd-12
    616 rabbit@juju-a2e49d-20-lxd-12
```

Stretch out services to new controller nodes (3->6) 60min:

```
$ for machine_id in $CTRL_POD2_NODES
do
  juju add-unit -n 1 --to lxd:$machine_id rabbitmq-server
done
$ juju-wait -v
```

Notice: check the nova compute service lists output to make sure no compute services
are in down status due to a possible race condition. In this case the following
services must be restarted:
- nova scheduler
- nova cloud controller
- nova computes


```
$ juju status rabbitmq-server | grep ^rabbitmq-server
rabbitmq-server   3.6.10   active      6  rabbitmq-server   jujucharms   97  ubuntu  
rabbitmq-server/0*       active    idle   18/lxd/16  10.15.147.106   5672/tcp       Unit is ready and clustered
rabbitmq-server/1        active    idle   19/lxd/12  10.15.147.102   5672/tcp       Unit is ready and clustered
rabbitmq-server/2        active    idle   20/lxd/12  10.15.147.166   5672/tcp       Unit is ready and clustered
rabbitmq-server/3        active    idle   65/lxd/7   10.15.147.126   5672/tcp       Unit is ready and clustered
rabbitmq-server/4        active    idle   66/lxd/7   10.15.147.136   5672/tcp       Unit is ready and clustered
rabbitmq-server/5        active    idle   67/lxd/7   10.15.147.78    5672/tcp       Unit is ready and clustered
```

Check the queue balance again:

```
$ juju run -u rabbitmq-server/leader -- rabbitmqctl list_queues -p openstack pid | sed -e 's/<\([^.]*\).*/\1/' | sort | uniq -c
      1 Listing queues
    605 rabbit@juju-a2e49d-18-lxd-16
    541 rabbit@juju-a2e49d-19-lxd-12
    476 rabbit@juju-a2e49d-20-lxd-12
     96 rabbit@juju-a2e49d-66-lxd-7
     48 rabbit@juju-a2e49d-67-lxd-7
```

Looks very unbalanced: 96 queues on 66/lxd/7, 48 on 67/lxd/7 and 65/lxd/7 doesn't
seem to be participating at all.

```
$ juju scp scripts/expansion/rebalance-queue-masters rabbitmq-server/0:
$ juju ssh rabbitmq-server/0
$ sudo ./rebalance-queue-masters -p openstack
$ exit
```

Remove non-leader units / 7min:

```
$ juju status rabbitmq-server | grep ^rabbitmq-server
rabbitmq-server   3.6.10   active      6  rabbitmq-server   jujucharms   97  ubuntu  
rabbitmq-server/0*       active    idle   18/lxd/16  10.15.147.106   5672/tcp       Unit is ready and clustered
rabbitmq-server/1        active    idle   19/lxd/12  10.15.147.102   5672/tcp       Unit is ready and clustered
rabbitmq-server/2        active    idle   20/lxd/12  10.15.147.166   5672/tcp       Unit is ready and clustered
rabbitmq-server/3        active    idle   65/lxd/7   10.15.147.126   5672/tcp       Unit is ready and clustered
rabbitmq-server/4        active    idle   66/lxd/7   10.15.147.136   5672/tcp       Unit is ready and clustered
rabbitmq-server/5        active    idle   67/lxd/7   10.15.147.78    5672/tcp       Unit is ready and clustered
```

```
$ juju remove-unit rabbitmq-server/1
$ juju remove-unit rabbitmq-server/2
$ juju-wait -v
```

As a second step, the leader should be removed safely, and a new leader
will be elected from the remaining units:

```
$ juju remove-unit rabbitmq-server/0
```

### Nagios problem: sta-cth03-rabbitmq-server-3 / NRPE: Unable to read output 

```
/usr/lib/nagios/plugins/check_nrpe -H 10.15.147.126 -c check_rabbitmq_queue -t 50
```

```
root@juju-a2e49d-65-lxd-7:/etc/nagios/nrpe.d# ll /usr/local/lib/nagios/plugins/check_rabbitmq_queues.py
-rwxr-xr-x 1 root root 3811 Jun 22 09:35 /usr/local/lib/nagios/plugins/check_rabbitmq_queues.py*
root@juju-a2e49d-65-lxd-7:/etc/nagios/nrpe.d# ll /var/lib/rabbitmq/data/juju-a2e49d-65-lxd-7_queue_stats.dat
-rw-r--r-- 1 root root 249199 Jun 29 18:20 /var/lib/rabbitmq/data/juju-a2e49d-65-lxd-7_queue_stats.dat

ll /var/lib/rabbitmq/ | grep dat
drwxr-x---  2 root     root        4096 Jun 29 18:20 data/


shoud be:
drwxr-xr-x  2 root     root        4096 Jun 29 18:20 data/

$ cat check_rabbitmq_queue.cfg 
# check rabbitmq_queue
# The following header was added automatically by juju
# Modifying it will affect nagios monitoring and alerting
# servicegroups: juju
command[check_rabbitmq_queue]=/usr/local/lib/nagios/plugins/check_rabbitmq_queues.py -c \* \* 100 200 /var/lib/rabbitmq/data/juju-a2e49d-65-lxd-7_queue_stats.dat
```

Workaround:

```
juju run -a rabbitmq-server "sudo chmod 0755 /var/lib/rabbitmq/data"
```

# Expand storage with pod2 nodes

Update bundle.yaml:

  - expected-ceph-osd count
  - add new machines
  - update num_units
  - add placements


Validate storage status:

Ceph juju units:
```
juju status ceph-mon | grep ^ceph-mon
ceph-mon          12.2.12  active      3  ceph-mon          jujucharms   44  ubuntu  
ceph-mon/2              active    idle   23/lxd/0  10.15.147.158                  Unit is ready and clustered
ceph-mon/4*             active    idle   25/lxd/0  10.15.147.77                   Unit is ready and clustered
ceph-mon/6              active    idle   58/lxd/0  10.15.147.64                   Unit is ready and clustered
```


```
$ juju ssh ceph-mon/4 'sudo ceph status'
Authorized uses only. All activity may be monitored and reported.
  cluster:
    id:     74335f40-f2d4-11ea-a9f0-00163e111dfd
    health: HEALTH_WARN
            too many PGs per OSD (274 > max 250)
 
  services:
    mon: 3 daemons, quorum juju-a2e49d-23-lxd-0,juju-a2e49d-25-lxd-0,juju-a2e49d-58-lxd-0
    mgr: juju-a2e49d-23-lxd-0(active), standbys: juju-a2e49d-25-lxd-0, juju-a2e49d-58-lxd-0
    osd: 18 osds: 18 up, 18 in
    rgw: 3 daemons active
 
  data:
    pools:   20 pools, 1644 pgs
    objects: 922.82k objects, 3.16TiB
    usage:   9.21TiB used, 89.0TiB / 98.2TiB avail
    pgs:     1644 active+clean
 
  io:
    client:   55.3MiB/s rd, 1.97MiB/s wr, 469op/s rd, 169op/s wr
 
Connection to 10.15.147.77 closed.


$ juju ssh ceph-mon/4 'sudo ceph osd tree'
Authorized uses only. All activity may be monitored and reported.
ID  CLASS WEIGHT   TYPE NAME                   STATUS REWEIGHT PRI-AFF 
 -1       98.24387 root default                                        
 -8       32.74796     rack az1                                        
-15       32.74796         host stanc01node007                         
  2   ssd  5.45799             osd.2               up  1.00000 1.00000 
  8   ssd  5.45799             osd.8               up  1.00000 1.00000 
 14   ssd  5.45799             osd.14              up  1.00000 1.00000 
 21   ssd  5.45799             osd.21              up  1.00000 1.00000 
 25   ssd  5.45799             osd.25              up  1.00000 1.00000 
 30   ssd  5.45799             osd.30              up  1.00000 1.00000 
 -4       32.74796     rack az2                                        
 -3       32.74796         host stanc01node009                         
  0   ssd  5.45799             osd.0               up  1.00000 1.00000 
  6   ssd  5.45799             osd.6               up  1.00000 1.00000 
 12   ssd  5.45799             osd.12              up  1.00000 1.00000 
 18   ssd  5.45799             osd.18              up  1.00000 1.00000 
 24   ssd  5.45799             osd.24              up  1.00000 1.00000 
 34   ssd  5.45799             osd.34              up  1.00000 1.00000 
-12       32.74796     rack az3                                        
-19       32.74796         host stanc01node011                         
  5   ssd  5.45799             osd.5               up  1.00000 1.00000 
 11   ssd  5.45799             osd.11              up  1.00000 1.00000 
 17   ssd  5.45799             osd.17              up  1.00000 1.00000 
 23   ssd  5.45799             osd.23              up  1.00000 1.00000 
 29   ssd  5.45799             osd.29              up  1.00000 1.00000 
 35   ssd  5.45799             osd.35              up  1.00000 1.00000 
Connection to 10.15.147.77 closed.
```


Execute ceph commands:

```
$ juju ssh ceph-mon/4 'sudo ceph osd set noin'
$ juju ssh ceph-mon/4 'sudo ceph osd set noscrub'
$ juju ssh ceph-mon/4 'sudo ceph osd set nodeep-scrub'
```


Check ceph status, the noin,noscrub,nodeep-scrub flags must be set:

```
$ juju ssh ceph-mon/4 'sudo ceph status'
Authorized uses only. All activity may be monitored and reported.
  cluster:
    id:     74335f40-f2d4-11ea-a9f0-00163e111dfd
    health: HEALTH_WARN
            noin,noscrub,nodeep-scrub flag(s) set
            too many PGs per OSD (274 > max 250)
 
  services:
    mon: 3 daemons, quorum juju-a2e49d-23-lxd-0,juju-a2e49d-25-lxd-0,juju-a2e49d-58-lxd-0
    mgr: juju-a2e49d-23-lxd-0(active), standbys: juju-a2e49d-25-lxd-0, juju-a2e49d-58-lxd-0
    osd: 18 osds: 18 up, 18 in
         flags noin,noscrub,nodeep-scrub
    rgw: 3 daemons active
 
  data:
    pools:   20 pools, 1644 pgs
    objects: 923.13k objects, 3.16TiB
    usage:   9.21TiB used, 89.0TiB / 98.2TiB avail
    pgs:     1644 active+clean
 
  io:
    client:   17.3KiB/s rd, 1.60MiB/s wr, 22op/s rd, 341op/s wr
 
Connection to 10.15.147.77 closed.
```



```
$ POD2_STORAGE_NODES=$(echo $(for i in $(cat config/nodes.yaml | grep "storage','pod2" -B 7 | grep stanc01node | sed 's/.$//'); do
  echo -n "$i|"
done) | sed 's/.$//')
$ echo $POD2_STORAGE_NODES
stanc01node049|stanc01node050|stanc01node051|stanc01node052|stanc01node053|stanc01node054
```

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo update-grub'
done
```

Reboot the storage nodes per AZ:

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | grep "az1" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo reboot'
done
$ juju-wait -v
```

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | grep "az2" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo reboot'
done
$ juju-wait -v
```

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | grep "az3" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo reboot'
done
$ juju-wait -v
```

```
$ juju ssh ceph-mon/4 'sudo ceph osd unset noin'
```

Check osd balance:

```
$ juju ssh ceph-mon/4 'sudo ceph osd df'
Authorized uses only. All activity may be monitored and reported.
ID CLASS WEIGHT  REWEIGHT SIZE    USE     AVAIL   %USE VAR  PGS 
 2   ssd 5.45799  1.00000 5.46TiB  527GiB 4.94TiB 9.43 1.00 274 
 8   ssd 5.45799  1.00000 5.46TiB  525GiB 4.95TiB 9.39 1.00 271 
14   ssd 5.45799  1.00000 5.46TiB  521GiB 4.95TiB 9.32 0.99 274 
21   ssd 5.45799  1.00000 5.46TiB  526GiB 4.94TiB 9.42 1.00 272 
25   ssd 5.45799  1.00000 5.46TiB  530GiB 4.94TiB 9.49 1.01 275 
30   ssd 5.45799  1.00000 5.46TiB  519GiB 4.95TiB 9.28 0.99 276 
 1   ssd 5.45799        0      0B      0B      0B    0    0   0 
 3   ssd 5.45799        0      0B      0B      0B    0    0   0 
 4   ssd 5.45799        0      0B      0B      0B    0    0   0 
 7   ssd 5.45799        0      0B      0B      0B    0    0   0 
 9   ssd 5.45799        0      0B      0B      0B    0    0   0 
10   ssd 5.45799        0      0B      0B      0B    0    0   0 
13   ssd 5.45799        0      0B      0B      0B    0    0   0 
16   ssd 5.45799        0      0B      0B      0B    0    0   0 
20   ssd 5.45799        0      0B      0B      0B    0    0   0 
26   ssd 5.45799        0      0B      0B      0B    0    0   0 
28   ssd 5.45799        0      0B      0B      0B    0    0   0 
32   ssd 5.45799        0      0B      0B      0B    0    0   0 
 0   ssd 5.45799  1.00000 5.46TiB  523GiB 4.95TiB 9.36 0.99 277 
 6   ssd 5.45799  1.00000 5.46TiB  528GiB 4.94TiB 9.44 1.00 277 
12   ssd 5.45799  1.00000 5.46TiB  528GiB 4.94TiB 9.44 1.00 274 
18   ssd 5.45799  1.00000 5.46TiB  531GiB 4.94TiB 9.50 1.01 274 
24   ssd 5.45799  1.00000 5.46TiB  527GiB 4.94TiB 9.43 1.00 272 
34   ssd 5.45799  1.00000 5.46TiB  519GiB 4.95TiB 9.28 0.99 271 
15   ssd 5.45799        0      0B      0B      0B    0    0   0 
19   ssd 5.45799        0      0B      0B      0B    0    0   0 
22   ssd 5.45799        0      0B      0B      0B    0    0   0 
27   ssd 5.45799        0      0B      0B      0B    0    0   0 
31   ssd 5.45799        0      0B      0B      0B    0    0   0 
33   ssd 5.45799        0      0B      0B      0B    0    0   0 
36   ssd 5.45799        0      0B      0B      0B    0    0   0 
37   ssd 5.45799        0      0B      0B      0B    0    0   0 
38   ssd 5.45799        0      0B      0B      0B    0    0   0 
39   ssd 5.45799        0      0B      0B      0B    0    0   0 
40   ssd 5.45799        0      0B      0B      0B    0    0   0 
43   ssd 5.45799        0      0B      0B      0B    0    0   0 
 5   ssd 5.45799  1.00000 5.46TiB  541GiB 4.93TiB 9.68 1.03 282 
11   ssd 5.45799  1.00000 5.46TiB  533GiB 4.94TiB 9.54 1.01 277 
17   ssd 5.45799  1.00000 5.46TiB  516GiB 4.95TiB 9.23 0.98 270 
23   ssd 5.45799  1.00000 5.46TiB  520GiB 4.95TiB 9.30 0.99 270 
29   ssd 5.45799  1.00000 5.46TiB  520GiB 4.95TiB 9.30 0.99 273 
35   ssd 5.45799  1.00000 5.46TiB  534GiB 4.94TiB 9.56 1.02 273 
41   ssd 5.45799        0      0B      0B      0B    0    0   0 
42   ssd 5.45799        0      0B      0B      0B    0    0   0 
45   ssd 5.45799        0      0B      0B      0B    0    0   0 
47   ssd 5.45799        0      0B      0B      0B    0    0   0 
49   ssd 5.45799        0      0B      0B      0B    0    0   0 
52   ssd 5.45799        0      0B      0B      0B    0    0   0 
44   ssd 5.45799        0      0B      0B      0B    0    0   0 
46   ssd 5.45799        0      0B      0B      0B    0    0   0 
48   ssd 5.45799        0      0B      0B      0B    0    0   0 
50   ssd 5.45799        0      0B      0B      0B    0    0   0 
51   ssd 5.45799        0      0B      0B      0B    0    0   0 
53   ssd 5.45799        0      0B      0B      0B    0    0   0 
                    TOTAL  295TiB 9.28TiB  285TiB 9.41          
MIN/MAX VAR: 0.98/1.03  STDDEV: 0.11
Connection to 10.15.147.77 closed.
```

Restart osds on new nodes:

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo systemctl restart ceph-osd@*'
done
```

Wait until just the noscrub, nodeep-scrub flags are set only:

```
$ juju ssh ceph-mon/4 'sudo ceph status'
Authorized uses only. All activity may be monitored and reported.
  cluster:
    id:     74335f40-f2d4-11ea-a9f0-00163e111dfd
    health: HEALTH_WARN
            noscrub,nodeep-scrub flag(s) set
```

Check the OSD balance again:

```
$ juju ssh ceph-mon/4 'sudo ceph osd df'
Authorized uses only. All activity may be monitored and reported.
ID CLASS WEIGHT  REWEIGHT SIZE    USE     AVAIL   %USE VAR  PGS 
 2   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.16 1.00  92 
 8   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.13 0.99  90 
14   ssd 5.45799  1.00000 5.46TiB  172GiB 5.29TiB 3.08 0.98  88 
21   ssd 5.45799  1.00000 5.46TiB  171GiB 5.29TiB 3.06 0.97  89 
25   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.16 1.00  91 
30   ssd 5.45799  1.00000 5.46TiB  171GiB 5.29TiB 3.07 0.97  90 
 1   ssd 5.45799  1.00000 5.46TiB  182GiB 5.28TiB 3.25 1.03  94 
 3   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.13 0.99  93 
 4   ssd 5.45799  1.00000 5.46TiB  172GiB 5.29TiB 3.08 0.98  93 
 7   ssd 5.45799  1.00000 5.46TiB  179GiB 5.28TiB 3.21 1.02  92 
 9   ssd 5.45799  1.00000 5.46TiB  183GiB 5.28TiB 3.27 1.04  91 
10   ssd 5.45799  1.00000 5.46TiB  178GiB 5.28TiB 3.19 1.01  93 
13   ssd 5.45799  1.00000 5.46TiB  184GiB 5.28TiB 3.29 1.05  94 
16   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.22 1.02  90 
20   ssd 5.45799  1.00000 5.46TiB  169GiB 5.29TiB 3.02 0.96  91 
26   ssd 5.45799  1.00000 5.46TiB  171GiB 5.29TiB 3.07 0.97  89 
28   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.14 1.00  93 
32   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  91 
 0   ssd 5.45799  1.00000 5.46TiB  166GiB 5.30TiB 2.98 0.95  92 
 6   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.22 1.02  95 
12   ssd 5.45799  1.00000 5.46TiB  177GiB 5.28TiB 3.17 1.01  93 
18   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.22 1.02  94 
24   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  92 
34   ssd 5.45799  1.00000 5.46TiB  172GiB 5.29TiB 3.08 0.98  89 
15   ssd 5.45799  1.00000 5.46TiB  178GiB 5.28TiB 3.18 1.01  91 
19   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  90 
22   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.12 0.99  92 
27   ssd 5.45799  1.00000 5.46TiB  174GiB 5.29TiB 3.11 0.99  90 
31   ssd 5.45799  1.00000 5.46TiB  182GiB 5.28TiB 3.25 1.03  92 
33   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  90 
36   ssd 5.45799  1.00000 5.46TiB  168GiB 5.29TiB 3.00 0.95  88 
37   ssd 5.45799  1.00000 5.46TiB  179GiB 5.28TiB 3.20 1.02  94 
38   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.14 1.00  91 
39   ssd 5.45799  1.00000 5.46TiB  177GiB 5.28TiB 3.17 1.01  92 
40   ssd 5.45799  1.00000 5.46TiB  177GiB 5.29TiB 3.16 1.00  90 
43   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.23 1.02  89 
 5   ssd 5.45799  1.00000 5.46TiB  181GiB 5.28TiB 3.23 1.03  98 
11   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  91 
17   ssd 5.45799  1.00000 5.46TiB  174GiB 5.29TiB 3.11 0.99  91 
23   ssd 5.45799  1.00000 5.46TiB  174GiB 5.29TiB 3.11 0.99  93 
29   ssd 5.45799  1.00000 5.46TiB  177GiB 5.29TiB 3.17 1.01  91 
35   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.21 1.02  92 
41   ssd 5.45799  1.00000 5.46TiB  177GiB 5.29TiB 3.16 1.00  92 
42   ssd 5.45799  1.00000 5.46TiB  179GiB 5.28TiB 3.21 1.02  93 
45   ssd 5.45799  1.00000 5.46TiB  173GiB 5.29TiB 3.09 0.98  92 
47   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  91 
49   ssd 5.45799  1.00000 5.46TiB  178GiB 5.28TiB 3.19 1.01  92 
52   ssd 5.45799  1.00000 5.46TiB  169GiB 5.29TiB 3.03 0.96  88 
44   ssd 5.45799  1.00000 5.46TiB  177GiB 5.28TiB 3.18 1.01  92 
46   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.15 1.00  91 
48   ssd 5.45799  1.00000 5.46TiB  175GiB 5.29TiB 3.13 0.99  88 
50   ssd 5.45799  1.00000 5.46TiB  180GiB 5.28TiB 3.22 1.02  92 
51   ssd 5.45799  1.00000 5.46TiB  176GiB 5.29TiB 3.14 1.00  90 
53   ssd 5.45799  1.00000 5.46TiB  170GiB 5.29TiB 3.04 0.97  87 
                    TOTAL  295TiB 9.28TiB  285TiB 3.15          
MIN/MAX VAR: 0.95/1.05  STDDEV: 0.07
Connection to 10.15.147.77 closed.
```

Enable scrub and nodeep-scrub again:

```
$ juju ssh ceph-mon/4 'sudo ceph osd unset noscrub'
$ juju ssh ceph-mon/4 'sudo ceph osd unset nodeep-scrub'
```

Check the ceph status again, health must have the HEALTH_OK state:

```
$ juju ssh ceph-mon/4 'sudo ceph status'
Authorized uses only. All activity may be monitored and reported.
  cluster:
    id:     74335f40-f2d4-11ea-a9f0-00163e111dfd
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum juju-a2e49d-23-lxd-0,juju-a2e49d-25-lxd-0,juju-a2e49d-58-lxd-0
    mgr: juju-a2e49d-23-lxd-0(active), standbys: juju-a2e49d-25-lxd-0, juju-a2e49d-58-lxd-0
    osd: 54 osds: 54 up, 54 in
    rgw: 3 daemons active
 
  data:
    pools:   20 pools, 1644 pgs
    objects: 926.33k objects, 3.17TiB
    usage:   9.28TiB used, 285TiB / 295TiB avail
    pgs:     1644 active+clean
 
  io:
    client:   176KiB/s rd, 3.29MiB/s wr, 296op/s rd, 1.71kop/s wr
 
Connection to 10.15.147.77 closed.
```


Nagios issue with CIS permission on tmpfs after rebooting ceph-osd nodes
Run on ALL ceph-osd units

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | awk {'print $1;'})
do
  echo 'chmod g+r /var/lib/ceph/osd/ceph-*/whoami; ls -l /var/lib/ceph/osd/ceph-*/whoami' | juju ssh $i sudo bash
done
```

Nagios error: Something went wrong reading the file: [Errno 13] Permission denied: '/var/lib/nagios/ceph-osd-checks' 

```
juju ssh ceph-osd/2 "sudo ls -alh /var/lib/nagios/ceph-osd-checks"
Authorized uses only. All activity may be monitored and reported.
-rw-rw-r-- 1 nagios nagios 208 Jun 29 18:02 /var/lib/nagios/ceph-osd-checks
Connection to 10.15.146.16 closed.
```

```
$ for i in $(juju machines | egrep "$POD2_STORAGE_NODES" | awk {'print $1;'})
do
  echo 'chmod 0664 /var/lib/nagios/ceph-osd-checks; chown nagios:nagios /var/lib/nagios/ceph-osd-checks; ls -alh /var/lib/nagios/ceph-osd-checks' | juju ssh $i sudo bash
done
```


# Add new compute nodes

## Deploy compute nodes

Update bundle.yaml

  - add new machines
  - update num_units
  - add placements

Deploy the updated bundle / 200min:

```
$ juju deploy ./config/bundle.yaml \
  --overlay config/overlays/ldap.yaml \
  --overlay config/overlays/hostnames.yaml \
  --overlay config/overlays/ssl.yaml \
  --overlay config/overlays/ovs.yaml \
  --overlay config/overlays/openstack_versioned_overlay.yaml

$ juju-wait -v
```

## Apply sysconfig settings

Collect pod2 compute nodes / 8min:

```
$ POD2_COMPUTE_NODES=$(echo $(for i in $(cat config/nodes.yaml | grep "ovs','pod2" -B 7 | grep stanc01node | sed 's/.$//'); do
  echo -n "$i|"
done) | sed 's/.$//')
$ echo $POD2_COMPUTE_NODES
```

```
$ for i in $(juju machines | egrep "$POD2_COMPUTE_NODES" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo update-grub'
done
```

Reboot the pod2 compute nodes:

```
$ for i in $(juju machines | egrep "$POD2_COMPUTE_NODES" | awk {'print $1;'})
do
  juju ssh -m openstack $i 'sudo reboot'
done
$ juju-wait -v
```

### Set host aggregates

`TODO:` a better script to collect host aggregate compute node ids.

`Notice:` current scripts are handling the last 2 digits, in case of pod3, minimum
3 digits will be required.

```
$ cat config/nodes.yaml | grep "ovs','pod2','az1'" -B 7 | grep stanc01node | cut -c13-
44:
55:
56:
61:
62:
67:
68:
73:
74:
79:
80:

44 55 56 61 62 67 68 73 74 79 80
```

```
$ cat config/nodes.yaml | grep "ovs','pod2','az2'" -B 7 | grep stanc01node | cut -c13-

46:
57:
58:
63:
64:
69:
70:
75:
76:
81:
82:
```

```
$ cat config/nodes.yaml | grep "ovs','pod2','az2'" -B 7 | grep stanc01node | cut -c13-

46 57 58 63 64 69 70 75 76 81 82

48:
59:
60:
65:
66:
71:
72:
77:
78:
83:
84:

48 59 60 65 66 71 72 77 78 83 84
```

Apply host aggregates:

```
$ scripts/post-deploy/14-pod2-xxxx.sh
```

# Tests

## Juju-lint

```
$ cd ~/deployment/stalbans-cth03-canonical/pod2-expansion-assets
$ juju status -o juju-status.json --format=json
$ juju-lint --config=../config/canonical-openstack-rules.yaml juju-status.json 2>juju-lint.txt
```

## Rally

Collect pod2 compute node ids:

```
$ POD2_COMPUTE_NODES=$(echo $(for i in $(cat config/nodes.yaml | grep "ovs','pod2" -B 7 | grep stanc01node | sed 's/.$//'); do
  echo -n "$i "
done))
echo $POD2_COMPUTE_NODES
```

```
source ./post-deploy/novarc
./scripts/post-deployment/13-create-rally-pod2-aggregates.sh
./scripts/post-deployment/13-create-rally-pod2-flavors.sh
```

```
$ openstack aggregate show rally -f value
None
2021-06-25T17:06:19.000000
False
None
['stanc01node044', 'stanc01node046', 'stanc01node048', 'stanc01node055', 'stanc01node056', 'stanc01node057', 'stanc01node058', 'stanc01node059', 'stanc01node060', 'stanc01node061', 'stanc01node062', 'stanc01node063', 'stanc01node064', 'stanc01node065', 'stanc01node066', 'stanc01node067', 'stanc01node068', 'stanc01node069', 'stanc01node070', 'stanc01node071', 'stanc01node072', 'stanc01node073', 'stanc01node074', 'stanc01node075', 'stanc01node076', 'stanc01node077', 'stanc01node078', 'stanc01node079', 'stanc01node080', 'stanc01node081', 'stanc01node082', 'stanc01node083', 'stanc01node084']
57
rally
{'host_aggregate': 'rally'}
None
```

```
$ openstack flavor list -f value | grep rally
1e755831-1668-4f6d-a287-63957f3c6356 rally.m1.small 2048 20 0 1 True
9188cd4b-0094-4a8c-a02e-16e98d54dcef rally.m1.xlarge 16384 160 0 8 True
a0fbf685-3105-4767-a830-87a00095ee12 rally.m1.large 8192 80 0 4 True
f8f5a8d8-99ce-416f-9619-fa60b0ccebe0 rally.m1.medium 4096 40 0 2 True
```

```
$ fcbtest.rallyinit
$ fcbtest.rally task start --task ./config/rally.yaml --task-args '{image_name: cirros, flavor_name: rally.m1.small, resize_flavor_name: rally.m1.large}'
```

If rally failed and did not clean the resources properly, the following resources might need to be removed:

```
for i in $(openstack project list -c Name | grep rally | awk '{print $2}'); do openstack project delete $i; done
for i in $(openstack user list -c Name | grep rally | awk '{print $2}'); do openstack user delete $i; done
for i in $(openstack server list -c Name | grep rally | awk '{print $2}'); do openstack server delete $i; done
openstack router list # It is easier to remove them in the GUI
for i in $(openstack network list -c Name | grep rally | awk '{print $2}'); do openstack network delete $i; done
```

### Issue: NeutronNetworks.create_and_delete_subnets rally test Unexpected error: No more IP addresses available on network bf60c822-f757-40f0-aa3b-1c8838742dc5. Neutron server returns request_ids: ['req-341e3e36-1169-470d-a742-d086c741d81c'] 

# Bond failover tests

```
$  POD2_NODES=$(echo $(for i in $(cat config/nodes.yaml | grep "pod2" -B 7 | grep stanc01node | sed 's/.$//'); do
echo -n "$i|"
done) | sed 's/.$//')
$ POD2_MACHINES=$(juju machines | egrep "$POD2_NODES" | awk {'print $1;'})
```

## test eno1

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno1up.out
grep -A1 -in eno1 bondfailovereno1up.out
```

### down the eno1 interfaces

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno1down.out
grep -A1 -in eno1 bondfailovereno1down.out
```

### back the eno1 interfaces

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno1up_back.out
grep -A1 -in eno1 bondfailovereno1up_back.out
```

## test eno2 interfaces

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno2up.out
grep -A1 -in eno2 bondfailovereno2up.out
```

### down the eno2 interfaces

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno2down.out
grep -A1 -in eno2 bondfailovereno2down.out
```

### back the eno2 interfaces

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond0'; done > bondfailovereno2up_back.out
grep -A1 -in eno2 bondfailovereno2up_back.out
```

bond1 set test [#1]

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond1'; done > bondfailover-bond1-up.out
egrep -A1 -in 'enp25s0f0' bondfailover-bond1-up.out
```

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond1'; done > bondfailover-bond1-down1.out
egrep -A1 -in 'enp25s0f0' bondfailover-bond1-down1.out
```

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond1'; done > bondfailover-bond1-up1.out
grep -A1 -in 'enp25s0f0' bondfailover-bond1-up1.out
```

bond1 set test [#2]

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond1'; done > bondfailover-bond1-down2.out
egrep -A1 -in 'enp25s0f1|enp94s0f1' bondfailover-bond1-down2.out
```

```
egrep -A1 -in 'enp25s0f1|enp94s0f1' bondfailover-bond1-down2.out | egrep -v 'enp25s0f1|enp94s0f1|down'
```

```
for i in $POD2_MACHINES; do juju ssh $i 'hostname; cat /proc/net/bonding/bond1'; done > bondfailover-bond1-up2
egrep -A1 -in 'enp25s0f1|enp94s0f1' bondfailover-bond1-up2.out
```

```
egrep -A1 -in 'enp25s0f1|enp94s0f1' bondfailover-bond1-up2.out | egrep -v 'enp25s0f1|enp94s0f1|up'
```
