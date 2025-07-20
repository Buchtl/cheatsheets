## Run NFS server directly on the host

Run `nfs-kernel-server` directly on your Ubuntu host, and **bind-mount** whatever directories you want from Docker:

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/nfs/shared
sudo chown 2000:2000 /srv/nfs/shared
```

`/etc/exports` (192.168.1.0/24 and 192.168.2.0/24 would be the allowed nfs-clients):

```
/srv/nfs/shared 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash) \
                192.168.2.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Then:

```bash
sudo exportfs -ra
```

## Setup UFW Firewall

Caution: the order of the rules matter -> the first rule that matches is applied by ufw

### Allow for 1 address
```
sudo ufw allow from 192.168.1.10 to any port 111 proto tcp
sudo ufw allow from 192.168.1.10 to any port 111 proto udp
sudo ufw allow from 192.168.1.10 to any port 2049 proto tcp
sudo ufw allow from 192.168.1.10 to any port 2049 proto udp
# NFSv3
sudo ufw allow from 192.168.1.10 to any port 20048 proto tcp
sudo ufw allow from 192.168.1.10 to any port 20048 proto udp
```

### Deny All
```
sudo ufw deny 111/tcp
sudo ufw deny 111/udp
sudo ufw deny 2049/tcp
sudo ufw deny 2049/udp
sudo ufw deny 20048/tcp
sudo ufw deny 20048/udp
```

### Reload
```
sudo ufw reload
```

### Check Status
```
sudo ufw reload
sudo ufw status verbose
sudo ufw status numbered
rpcinfo -p

# to monitor incoming requests e.g.
sudo tcpdump -n -i enp0s3 port 2049
```

## Mount nfs-share in Docker Compose/Swarm


### Example NFS-Client
```
services:
  nfs-client:
    build: .
    image: ubuntu:24.04
    user: "2000:2000"
    container_name: nfs-client
    command: "tail -f /dev/null"
    volumes:
      - nfs-filestorage:/filestorge
volumes:
  nfs-filestorage:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=${IP_NFS_SERVER},nfsvers=4,hard,rw"
      device: ":/srv/nfs/shared"
```
