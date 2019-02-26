
GlusterFS role
==============
- Create LVM thin volume and filesystem (XFS) on `storage_devices`
- Configure firewalld for GlusterFS
- Install glusterfs
- Do not manage Gluster volume :warning: Manuel Step :warning:
- Active glusterfs volumes
- Mount glusterfs volumes



Installation
------------

1. play role

2. Manually manage the volume   
On one gluster server, connect peers and create volumes
```bash
# Create pool of trusted servers
gluster peer probe 192.168.103.32
gluster peer probe 192.168.103.33
gluster peer probe 192.168.103.34
gluster peer probe 192.168.103.35
gluster peer probe 192.168.103.36
# Volume /scratch (distributed)
gluster volume create scratch \
192.168.103.32:/bricks/scratch/brick1/brick \
192.168.103.33:/bricks/scratch/brick1/brick \
192.168.103.34:/bricks/scratch/brick1/brick \
192.168.103.35:/bricks/scratch/brick1/brick \
192.168.103.36:/bricks/scratch/brick1/brick
# Volume /data (replicated-distributed 2 réplicas + arbiter)
gluster volume create data replica 3 arbiter 1 \
192.168.103.32:/bricks/data/brick1/brick 192.168.103.33:/bricks/data/brick2/brick 192.168.103.34:/bricks/data/arbiter/brick \
192.168.103.33:/bricks/data/brick1/brick 192.168.103.34:/bricks/data/brick2/brick 192.168.103.35:/bricks/data/arbiter/brick \
192.168.103.34:/bricks/data/brick1/brick 192.168.103.35:/bricks/data/brick2/brick 192.168.103.36:/bricks/data/arbiter/brick \
192.168.103.35:/bricks/data/brick1/brick 192.168.103.36:/bricks/data/brick2/brick 192.168.103.32:/bricks/data/arbiter/brick \
192.168.103.36:/bricks/data/brick1/brick 192.168.103.32:/bricks/data/brick2/brick 192.168.103.33:/bricks/data/arbiter/brick
```

3. Play role again



Sources
-------
Largement pompé de:

* zaxos/glusterfs-ansible-role  
George Zachariadis  
Licence GPLv2  
https://github.com/zaxos/glusterfs-ansible-role  
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-_zaxos.glusterfs--ansible--role-blue.svg)](https://galaxy.ansible.com/zaxos/glusterfs-ansible-role/)  


* module ansible gluster_volume  
http://docs.ansible.com/ansible/latest/gluster_volume_module.html  

* modules LVM ansible  
http://docs.ansible.com/ansible/latest/lvg_module.html  
http://docs.ansible.com/ansible/latest/lvol_module.html  



Role Variables
--------------
- `glusterfs_version` = GlusterFS version to be installed using CentOS Storage SIG Packages. Default is the version "4.1".
- `glusterfs_vg` = List of VG: `vgname`, `storage_devices` (List of comma-separated devices)
- `glusterfs_thinpool` = List of thinpool: `vgname`, `poolname`, `poolsize` (avoid floating value)
- `glusterfs_lv` = List of LVM thin logical volume: `lvname`, `vgname`, `poolname` (optional), `lvsize`, `mountpath`
- `glusterfs_nodes` = List of nodes in GlusterFS cluster. By default this list is populated by the defined group in inventory
- `glusterfs_volumes` = List of volumes: `volume` (name), `state`, `options`, `mount`: server, options, path,...
- `firewalld_enabled`: Configure firewall

```yaml 
---
# Configure Firewall
firewalld_enabled: true

# Configure GlusterFS
glusterfs_volumes:
- volume: volume1  # Replicated
  state: present       
  replica: 3
  mount:
    path: "/mnt/volume1"
        
- volume: volume2  # Distributed in 2 nodes
  state: present
  nodes:
    - node2.glusterfs.example
    - node3.glusterfs.example
  options:
    performance.cache-size: 256MB
  mount:
    server: "node2.glusterfs.example"
    options: "defaults,_netdev,backupvolfile-server=node3.glusterfs.example"
    path: "/mnt/volume2"
    owner: exampleuser		# Default: root
    group: examplegroup		# Default: root
    mode: "0770"		# Default: 0755
```


Mount options
-------------

See:
- http://docs.gluster.org/en/latest/Administrator%20Guide/Setting%20Up%20Clients/#mounting-volumes
- man mount.glusterfs

Notes sur le montage:
- The server specified in the mount command is only used to fetch the gluster configuration volfile describing the volume name. Subsequently, the client will communicate directly with the servers mentioned in the volfile (which might not even include the one used for mount).
- L'option backupvolfile-server permets de sépcifier une autre server /volume de backup.





Full example
------------
```yaml
---

glusterfs_version: "4.1"


glusterfs_vg:
  - vgname: "vgscratch"
    storage_devices: "/dev/sdb"
  - vgname: "vgdata"
    storage_devices: "/dev/sdc"


glusterfs_thinpool:
   - vgname: "vgdata"
     poolname: "thinpooldata"
     poolsize: "9G"


glusterfs_lv:
  - lvname: "scratch-brick1"
    vgname: "vgscratch"
    lvsize: "9G"
    mountpath: "/bricks/scratch/brick1"
  - lvname: "data-brick1"
    vgname: "vgdata"
    poolname: "thinpooldata"
    lvsize: "4G"
    mountpath: "/bricks/data/brick1"
  - lvname: "data-brick2"
    vgname: "vgdata"
    poolname: "thinpooldata"
    lvsize: "4G"
    mountpath: "/bricks/data/brick2"
  - lvname: "data-arbiter"
    vgname: "vgdata"
    poolname: "thinpooldata"
    lvsize: "1G"
    mountpath: "/bricks/data/arbiter"


glusterfs_volumes:

  # RAID-0 sur ssd, distributed
  - volume: scratch
    state: present
    mount:
      # http://docs.gluster.org/en/latest/Administrator%20Guide/Setting%20Up%20Clients/#mounting-volumes
      # man mount.glusterfs
      server: "192.168.103.32"
      options: "defaults,_netdev"
      path: "/shared/scratch"

  # RAID-6 sur disques SAS, replicated-distributed (2 réplicas + arbiter)
  - volume: data
    state: present
    mount:
      # http://docs.gluster.org/en/latest/Administrator%20Guide/Setting%20Up%20Clients/#mounting-volumes
      # man mount.glusterfs
      server: "192.168.103.32"
      options: "defaults,_netdev"
      path: "/shared/data"


```

