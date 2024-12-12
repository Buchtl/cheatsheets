# Run Once In Dev
When using Ansible with a setup where three logical servers are on the same physical machine during development, you can use a combination of strategies to ensure tasks and roles are executed only once. Here are a few approaches:

---

### **Option 1: Use Host Facts or Variables to Identify Single Physical Machine**
Set a unique variable or fact on the physical machine and use a condition to execute tasks or roles only if the condition matches.

#### Example Inventory
```yaml
all:
  children:
    logical_servers:
      hosts:
        server1:
          ansible_host: 127.0.0.1
          is_physical: true
        server2:
          ansible_host: 127.0.0.1
        server3:
          ansible_host: 127.0.0.1
```

#### Playbook with Conditional Execution
```yaml
- name: Ensure tasks execute only once
  hosts: logical_servers
  gather_facts: yes
  tasks:
    - name: Run task only if physical server
      debug:
        msg: "This task runs only once on the physical server"
      when: inventory_hostname == "server1"
```

Here, tasks will only execute for `server1`, preventing redundant executions for `server2` and `server3`.

---

### **Option 2: Use the `run_once` Directive**
The `run_once` directive ensures a task runs only once, regardless of how many hosts are in the inventory.

#### Playbook Example
```yaml
- name: Ensure tasks execute only once
  hosts: logical_servers
  tasks:
    - name: This task runs only once
      debug:
        msg: "This task runs once for all logical servers"
      run_once: true
```

This approach is useful when the task doesn't depend on which host executes it, but should apply globally.

---

### **Option 3: Group Variables to Designate a Primary Server**
Use a group variable to designate one host as the "primary" host for tasks.

#### Inventory
```yaml
all:
  children:
    logical_servers:
      hosts:
        server1:
          ansible_host: 127.0.0.1
        server2:
          ansible_host: 127.0.0.1
        server3:
          ansible_host: 127.0.0.1
      vars:
        primary_server: server1
```

#### Playbook Example
```yaml
- name: Execute tasks only on primary server
  hosts: logical_servers
  tasks:
    - name: This task runs only on the primary server
      debug:
        msg: "Running on the primary server"
      when: inventory_hostname == primary_server
```

---

### **Option 4: Deduplicate Roles Using `delegate_to`**
Run the role on only one host and delegate its execution to the primary server.

#### Playbook Example
```yaml
- name: Run roles only once
  hosts: logical_servers
  tasks:
    - name: Execute role on primary server
      include_role:
        name: your_role
      delegate_to: server1
      run_once: true
```

---

### **Best Practice Recommendation**
- During development, use `run_once` for general tasks or designate a "primary" server in your inventory with group variables.
- For production, ensure all logical servers are treated independently to reflect the real-world setup.

Would you like a deeper dive into any of these methods?

# SSH Keys
To run Ansible using SSH keys instead of an SSH login with a password, follow these steps:

### 1. **Generate SSH Key Pair**
If you don’t already have an SSH key pair, generate one:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Save the key pair in the default location (`~/.ssh/id_rsa`).

### 2. **Copy the Public Key to the Target Hosts**
Transfer your public key (`~/.ssh/id_rsa.pub`) to the target hosts:

```bash
ssh-copy-id -i ~/.ssh/id_rsa user@target_host
```

Replace `user` with the username and `target_host` with the hostname or IP address of your target machine.

Alternatively, you can manually append the public key to the `~/.ssh/authorized_keys` file on the target host.

### 3. **Configure Ansible Inventory**
Ensure your Ansible inventory file (`inventory` or `hosts`) lists the target hosts. For example:

```ini
[target_group]
target_host ansible_user=user
```

Replace `target_host` with the hostname or IP of the target machine and `user` with the SSH username.

### 4. **Specify the Private Key in Ansible**
You can specify the private key in multiple ways:

#### Option A: **Command Line**
Pass the private key using the `--private-key` option:

```bash
ansible -i inventory target_group -m ping --private-key ~/.ssh/id_rsa
```

#### Option B: **ansible.cfg**
Edit your `ansible.cfg` file to include the path to the private key:

```ini
[defaults]
inventory = inventory
remote_user = user
private_key_file = ~/.ssh/id_rsa
```

#### Option C: **Playbook-Specific Configuration**
In your playbook YAML file, use the `ansible_ssh_private_key_file` variable:

```yaml
- hosts: target_group
  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  tasks:
    - name: Ping the target
      ping:
```

### 5. **Test the Configuration**
Verify the setup by pinging the target hosts:

```bash
ansible -i inventory target_group -m ping
```

If configured correctly, Ansible should connect to the target hosts without prompting for a password.

# Init SSH
When connecting to a new server for the first time using SSH, the server's host key is not recognized by your local machine. This can cause Ansible or SSH commands to fail with a message like:

```
The authenticity of host 'host.example.com (1.2.3.4)' can't be established.
```

To fix this issue, you can handle the host key verification in one of the following ways:

---

### **1. Add the Host Key to Known Hosts**
Run an SSH command manually to add the server's host key to your `~/.ssh/known_hosts` file:

```bash
ssh user@target_host
```

Type `yes` when prompted to accept the server's host key. Once accepted, you can run Ansible without issues.

---

### **2. Disable Host Key Checking (Temporary Solution)**
You can disable host key checking in your Ansible commands or configuration, though this is less secure.

#### A. Command Line
Add the `-e ansible_ssh_extra_args` option:

```bash
ansible -i inventory target_group -m ping -e "ansible_ssh_extra_args='-o StrictHostKeyChecking=no'"
```

#### B. ansible.cfg
Modify your `ansible.cfg` file to include:

```ini
[defaults]
host_key_checking = False
```

#### C. Environment Variable
Set the environment variable to disable host key checking:

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
ansible -i inventory target_group -m ping
```

---

### **3. Automatically Accept Host Keys (Secure & Automated)**
Use the `ssh-keyscan` command to gather host keys for the new server and append them to your `~/.ssh/known_hosts` file.

```bash
ssh-keyscan -H target_host >> ~/.ssh/known_hosts
```

To automate this for multiple hosts in your inventory:

```bash
for host in $(awk '/^[^#]/ {print $1}' inventory); do ssh-keyscan -H $host >> ~/.ssh/known_hosts; done
```

This securely adds the host keys without disabling host key checking.

---

### **4. Use `add_host` Module in Ansible**
If you need to accept keys dynamically during playbook execution, you can use the `add_host` module in a task:

```yaml
- name: Add host keys to known_hosts
  hosts: localhost
  tasks:
    - name: Scan and add host key
      command: ssh-keyscan -H {{ inventory_hostname }} >> ~/.ssh/known_hosts
```

---

### **5. Combine with Ansible Automation**
You can pre-configure SSH keys and known hosts using Ansible itself:

```yaml
- hosts: localhost
  tasks:
    - name: Add server to known_hosts
      shell: ssh-keyscan -H {{ inventory_hostname }} >> ~/.ssh/known_hosts
      delegate_to: localhost
```

After this task runs, the new server's host key will be trusted, and subsequent SSH connections will work.

By handling the host key as shown above, you can avoid errors while maintaining security.
