## Run NFS server directly on the host

Run `nfs-kernel-server` directly on your Ubuntu host, and **bind-mount** whatever directories you want from Docker:

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/nfs/export
sudo chown nobody:nogroup /srv/nfs/export
```

`/etc/exports`:

```
/srv/nfs/export 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Then:

```bash
sudo exportfs -ra
```

## Setup UFW Firewall

### Deny All
```
sudo ufw deny 111/tcp
sudo ufw deny 111/udp
sudo ufw deny 2049/tcp
sudo ufw deny 2049/udp
sudo ufw deny 20048/tcp
sudo ufw deny 20048/udp
```
### Allow for 1 address
```
sudo ufw allow from 192.168.1.10 to any port 111 proto tcp
sudo ufw allow from 192.168.1.10 to any port 111 proto udp
sudo ufw allow from 192.168.1.10 to any port 2049 proto tcp
sudo ufw allow from 192.168.1.10 to any port 2049 proto udp
sudo ufw allow from 192.168.1.10 to any port 20048 proto tcp
sudo ufw allow from 192.168.1.10 to any port 20048 proto udp
```

### Reload
```
sudo ufw reload
```

### Check Status
```
sudo ufw reload
sudo ufw status verbose
rpcinfo -p
```

## Mount nfs-share in Docker Compose/Swarm

### Dockerfile for example client
```
FROM ubuntu:24.04

RUN groupadd -g 2000 nfs && \
    useradd -rm -d /home/nfs -s /bin/bash -u 2000 nfs -g nfs

USER nfs

CMD bash -c "tail -f /dev/null"

```

### Starting the service with mounted nfs-share
```
services:
  nfs-client:
    build: .
    image: nfs-client:01
    container_name: nfs-client
    volumes:
      - nfs-filestorage:/filestorge
volumes:
  nfs-filestorage:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=192.168.2.142,nfsvers=4,hard,rw"
      device: ":/srv/nfs/shared"
```
