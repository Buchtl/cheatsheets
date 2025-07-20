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
**Explanation**
* **`rw`**
  Clients in the allowed IP range can **read and write** to the shared directory.

* **`sync`**
  The NFS server **synchronously writes changes to disk before replying** to the client. This ensures data integrity but may reduce performance compared to async.

* **`no_subtree_check`**
  Disables subtree checking. Without it, if the exported directory is a subdirectory of a larger filesystem, the server verifies that a file being accessed is inside the exported subtree. This check can cause performance overhead or errors during renames, so `no_subtree_check` improves performance and reliability.

* **`no_root_squash`**
  Normally, NFS maps remote root users to an anonymous user (like `nobody`) for security.
  `no_root_squash` disables this behavior and lets remote root users have root privileges on the exported filesystem. This can be a security risk if clients are untrusted but is useful when root access is needed over NFS.


**Then:**

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

## Mount nfs-share "bare metal"
```
sudo mkdir /mnt/nfs/shared
sudo mount ${IP_NFS_SERVER}:/srv/nfs/shared /mnt/nfs/shared
```

## Mount nfs-share in Docker Compose/Swarm

### Example NFS-Client
#### Docker Compse
```
services:
  nfs-client:
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
#### Docker Swarm
```
version: "3.9"  # Docker Swarm requires version 3.x
services:
  nfs-client:
    image: ubuntu:24.04
    user: "2000:2000"
    command: >
      sh -c "env | grep IP_NFS_SERVER && tail -f /dev/null"
    environment:
      - IP_NFS_SERVER=${IP_NFS_SERVER}
    volumes:
      - nfs-filestorage:/filestorge
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
volumes:
  nfs-filestorage:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=${IP_NFS_SERVER},nfsvers=4,hard"
      device: ":/srv/nfs/shared"
```
Script to deploy stack
```
#!/bin/bash
IP=$1
STACK_NAME=nfs-client

docker swarm init --advertise-addr $IP
# to include .env: <(docker-compose config)
docker stack deploy -c <(docker-compose config) $STACK_NAME
docker stack ls
docker stack services $STACK_NAME
docker stack ps $STACK_NAME
```
