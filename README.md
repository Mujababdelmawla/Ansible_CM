# Ansible Practical — Nginx on AWS EC2

## What this practical does
Uses Ansible to automatically install and start Nginx on two remote AWS EC2 instances — from a single playbook, with zero manual SSH into the managed nodes.

---

## Infrastructure

| Role | Name | IP |
|---|---|---|
| Control node | Your local / master EC2 | (Ansible installed here) |
| Managed node 1 | ansible-node-01 | 13.200.249.149 |
| Managed node 2 | ansible-node-02 | 3.108.190.170 |

---

## Project structure

```
ansible-practical/
├── inventory.ini          # Defines managed nodes and SSH credentials
├── nginx.yml              # Playbook — installs and starts Nginx
└── ansible-key-01.pem    # SSH private key for EC2 access
```

---

## inventory.ini

```ini
[web]
ansible-node-01 ansible_host=13.200.249.149
ansible-node-02 ansible_host=3.108.190.170

[web:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file="./ansible-key-01.pem"
```

**What each part means:**
- `[web]` — group name; used in the playbook as `hosts: web`
- `ansible_host` — the public IP of each EC2 instance
- `ansible_user=ubuntu` — default user for Ubuntu EC2 AMIs
- `ansible_ssh_private_key_file` — path to your `.pem` key for SSH auth

---

## nginx.yml

```yaml
---
- name: configure nginx
  hosts: web
  become: yes

  tasks:
    - name: installing nginx
      apt:
        name: nginx
        state: present

    - name: starting nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

**What each part means:**
- `hosts: web` — targets the `[web]` group from inventory.ini
- `become: yes` — runs all tasks as sudo (root privileges)
- `state: present` — installs nginx only if not already installed (idempotent)
- `state: started` — ensures nginx is running
- `enabled: yes` — ensures nginx starts automatically on reboot ...done 

---

## Commands

### 1. Disable host key checking (required for cloud EC2)
```bash
export ANSIBLE_HOST_KEY_CHECKING=false
```
> Prevents Ansible from prompting "yes/no" for each new SSH connection. Required because EC2 instances are new hosts not in your known_hosts file.

### 2. Test connectivity
```bash
ansible web -i inventory.ini -m ping
```
> Expected output: `SUCCESS` with `ping: pong` for both nodes.

### 3. Run the playbook
```bash
ansible-playbook -i inventory.ini nginx.yml
```
> Expected output: `ok=2 changed=2` for both nodes on first run.  
> On re-run: `ok=2 changed=0` — idempotency in action.

---

## Key concepts demonstrated

| Concept | Where it appears |
|---|---|
| Agentless | No software installed on EC2 nodes — SSH only |
| Inventory | `inventory.ini` groups and variables |
| Declarative config | `state: present` not `apt install nginx` |
| Idempotency | Re-running playbook makes no changes |
| Privilege escalation | `become: yes` for sudo access |
| Ad-hoc command | `ansible web -m ping` |
| Playbook | `nginx.yml` — full automation in YAML |

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Permission denied (publickey) | Check `.pem` file path and permissions: `chmod 400 ansible-key-01.pem` |
| Host key verification failed | Run `export ANSIBLE_HOST_KEY_CHECKING=false` |
| Python not found on node | Ensure Ubuntu AMI has Python 3: `sudo apt install python3` |
| Nginx not accessible in browser | Check EC2 security group — port 80 must be open |