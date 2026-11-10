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