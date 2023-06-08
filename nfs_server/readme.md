### Simple NFSv3 server on Ubuntu 22 (jammy)
# Process overview
- Make NFS export directory if it is not present;
- Install NFS packages (nfs-kernel-server and nfs-common);
- Make changes in /etc/nfs.conf to enable only NFSv3. That's why this role is suitable only for Ubuntu 22 and later, cause in previous releases such configurations were modified in /etc/default/nfs-kernel-server. Restart nfs-kernel-server in case there were any changes in /etc/nfs.conf;
- Enable and start nfs-kernel-server;
- Make /etc/exports from template and reload NFS via exportfs in case there were any changes in /etc/exports.
# Variables
There is only single variable available.<br/>
Example:
```
nfs_server_exports:
  - path: /vol1/export1
    dir_permissions: "0777"
    clients: "10.215.0.0/16"
    read_write: rw
    sync: sync
    subtree: no_subtree_check
    root_squash: no_root_squash
  - path: /vol1/export2
    dir_permissions: "0644"
    clients: "*"
    read_write: rw
    sync: async
    subtree: no_subtree_check
    root_squash: no_root_squash
```
Available options:
- path - a directory for an NFS-export;
- dir_permissions - directory permissions (4-digit or rwx format);
- clients - IP-addrs or hostnames which will have an access to the particular NFS-export, e.g. "*", "192.168.0.0/24"
- read_write - ro/rw;
- sync - sync or async. Sync means NFS-server acknowledges data only after flushing (destaging) it on a disk, async - NFS-server acknowledges data immediately (before commiting it on a disk), there is a data loss risk in case of no BBU;
- subtree - subtree_check or no_subtree_check, see https://linux.die.net/man/5/exports;
- root_squash - no_root_squash or root_squash, https://linux.die.net/man/5/exports.