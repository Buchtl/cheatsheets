**Workers first, then the manager.**

With only one manager, this is the safer order for a general `apt upgrade` (and any required reboot).

### Why this order
- The single manager is a single point of failure for the control plane. If you take it down first, you temporarily lose the ability to drain nodes, schedule tasks, or manage the swarm.
- Existing containers on workers keep running even while the manager is offline. You just can’t make changes until it comes back.
- You can safely drain workers one by one from the manager, upgrade/reboot them, then bring them back before touching the manager.

### Recommended procedure
Do this **one node at a time**:

1. **On the manager**, drain the next worker:
   ```bash
   docker node update --availability drain <worker-hostname-or-id>
   ```
2. Wait until the tasks have moved off it (`docker node ps <node>` or `docker service ps ...`).
3. On that worker, run the upgrade:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   # optionally: sudo apt full-upgrade -y
   ```
4. Reboot if required (`[ -f /var/run/reboot-required ] && sudo reboot` or check `needrestart`).
5. After the node is back and healthy, activate it again from the manager:
   ```bash
   docker node update --availability active <worker-hostname-or-id>
   ```
6. Repeat for the remaining workers.
7. **Finally** drain and upgrade the manager the same way (it is also a worker by default unless you already drained it).

### Extra tips
- Prefer a maintenance window, especially if kernel or Docker packages are involved.
- If Docker itself is being upgraded, the same “one node at a time, workers first” approach still applies.
- After everything is done, check `docker node ls` and the status of your services.
- Long-term: a single manager is fragile. Consider promoting one or two workers to managers (odd number, usually 3) for better resilience.

This keeps the control plane available as long as possible while you roll through the workers.

**No — reactivate each worker as soon as it is healthy.**

Do **not** leave the workers drained until the manager is finished. Drain only the node you are currently upgrading, then bring it back online before moving to the next one.

### Correct flow
1. Drain **one** worker  
2. Upgrade + reboot it  
3. Wait until it is back and healthy  
4. Set it back to `active`  
5. Repeat for the next worker  
6. Finally do the manager the same way

### Why
- Leaving all workers drained removes almost all capacity for running containers.  
- Once a worker is upgraded and active again, the manager can immediately schedule tasks onto it.  
- This keeps the cluster usable while you work through the nodes.  
- Only the node you are touching should be drained at any moment.

### Quick commands reminder
```bash
# From the manager
docker node update --availability drain <node>
# ... upgrade/reboot the node ...
docker node update --availability active <node>
```

After each reactivation, confirm with `docker node ls` that the node is `Ready` and `Active` before continuing.

**Updating the manager (last step)**

Because this is your only manager, the control plane will be briefly unavailable while it is drained/rebooting. Existing containers on the workers continue running, but you cannot schedule new tasks or change the swarm until the manager is back.

### Steps

1. **Drain the manager** (from the manager itself, while it is still up):
   ```bash
   docker node update --availability drain $(hostname)
   # or use the node ID/hostname shown by docker node ls
   ```
   Wait until all tasks have moved off it:
   ```bash
   docker node ps $(hostname)
   # or
   docker service ls
   docker service ps <service-name>
   ```

2. **Run the upgrade** on the manager:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   # if packages were held back:
   # sudo apt full-upgrade -y
   ```

3. **Reboot** if required:
   ```bash
   [ -f /var/run/reboot-required ] && sudo reboot
   # or check with: needrestart -r a   (if installed)
   ```

4. After the node is back online, **reactivate it**:
   ```bash
   docker node update --availability active $(hostname)
   ```

5. **Verify**:
   ```bash
   docker node ls
   docker service ls
   docker service ps <your-services>
   ```

### Notes specific to the single manager
- While the manager is down, the swarm keeps running the already-scheduled tasks on the workers, but you lose management capability until it returns.
- Make sure at least some workers are already upgraded and `Active` before you start on the manager, so they can take the load.
- If the manager fails to rejoin the swarm after reboot (rare), the usual recovery is:
  ```bash
  docker swarm init --force-new-cluster --advertise-addr <manager-ip>
  ```
  (only as a last resort; workers will need to rejoin).

That’s the complete sequence: workers one-by-one → then the manager.


**Yes — drain the manager if it is currently Active and running tasks.**

Official Docker docs and standard practice treat the manager the same as any other node for maintenance: drain it first so its tasks are cleanly moved to the workers while the manager is still able to perform the rescheduling.

### What drain does (and does not do)
- It **stops** the manager from receiving new tasks and **migrates** existing ones to active workers.
- It does **not** demote the node or remove its manager role. It remains the leader/manager the whole time.
- Once the node is back, you simply set it to `active` again.

### Exact steps for the manager
```bash
# 1. Drain it (while it is still up)
docker node update --availability drain $(hostname)

# 2. Confirm tasks have left
docker node ps $(hostname)

# 3. Upgrade
sudo apt update && sudo apt upgrade -y

# 4. Reboot if needed
sudo reboot

# 5. After it is back and reachable, reactivate
docker node update --availability active $(hostname)

# 6. Verify
docker node ls
```

### When you can skip the drain
Only if the manager is **already** in `Drain` state (i.e. you previously made it manager-only and it is not running any service tasks). In that case you can just upgrade and reboot.

Some sources say you can simply reboot without draining and rely on restart policies, but that is less clean and can cause brief task interruptions. Draining first is the safer and recommended approach, especially with a single manager.


**Das ist ein bekanntes Problem mit neueren Docker-/gRPC-Versionen und älteren Swarm-Clustern.**

Neuere Docker-Versionen (mit gRPC-go ≥ 1.67) erzwingen ALPN (Application-Layer Protocol Negotiation, speziell das `h2`-Protokoll für HTTP/2) beim TLS-Handshake. Ältere Swarm-Manager werben dieses Protokoll in ihrer TLS-Konfiguration oft nicht korrekt an. Deshalb scheitert der Join mit einem Handshake-Fehler, der „alpn“ oder „missing selected ALPN property“ enthält.

### Lösung auf der neuen VM (Workaround)

Setze die Umgebungsvariable `GRPC_ENFORCE_ALPN_ENABLED=false` für den Docker-Daemon:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/systemd/system/docker.service.d/alpn-workaround.conf > /dev/null <<EOF
[Service]
Environment="GRPC_ENFORCE_ALPN_ENABLED=false"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

Danach erneut joinen:

```bash
docker swarm join --token <TOKEN> <MANAGER-IP>:2377
```

(Optional `--advertise-addr <IP-der-neuen-VM>` mitgeben, falls die automatische Erkennung nicht stimmt.)

### Weitere Checks

1. **Docker-Versionen vergleichen**
   ```bash
   docker version
   ```
   Auf Manager und neuer VM. Ideal sind möglichst gleiche oder zumindest kompatible Versionen. Der offizielle Fix in SwarmKit (NextProtos/`h2` in der Server-TLS-Config) kam Anfang 2025.

2. **Ports öffnen** (auf allen Nodes und Firewalls):
   - 2377/tcp (Swarm-Management)
   - 7946/tcp + 7946/udp (Node-Kommunikation)
   - 4789/udp (VXLAN-Overlay)

3. **Verbindung testen**
   ```bash
   telnet <MANAGER-IP> 2377
   # oder
   nc -zv <MANAGER-IP> 2377
   ```

4. **Logs anschauen**
   ```bash
   journalctl -u docker -f
   # oder
   docker info
   ```

### Langfristig

- Den alten Swarm-Cluster auf eine neuere Docker-Version upgraden (dann ist der Workaround auf neuen Nodes nicht mehr nötig).
- Alternativ die neue VM vorerst auf eine ältere Docker-Version pinnen, die noch kompatibel ist.

Der Workaround mit `GRPC_ENFORCE_ALPN_ENABLED=false` ist der übliche und gut dokumentierte Weg für genau diesen ALPN-Handshake-Fehler zwischen alten und neuen Swarm-Nodes.