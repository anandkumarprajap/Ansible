# Ansible Master--Node Setup on AWS EC2

> **Security warning:** A private SSH key was pasted into the chat text.
> Treat that key as compromised. In AWS, create a replacement key pair
> and rebuild/reconfigure access for the instances, then stop using the
> exposed key. Never put private-key contents in notes, GitHub, chat, or
> screenshots. This README intentionally does not include the key.

## 1. Goal

Use one Ubuntu EC2 instance as the **Ansible control node (master)** to
configure two Ubuntu EC2 managed nodes:

-   `node-1` --- install common packages and optionally deploy Devboard
-   `node-2` --- install common packages and optionally deploy Devboard
-   Ansible master --- runs Ansible commands and playbooks; it does not
    need to run the application

## 2. Architecture and flow diagrams

### Infrastructure and control flow

``` mermaid
flowchart TD
    A[Windows PC / PowerShell] -->|SSH with private key| M[Ansible Master EC2<br/>Ubuntu]
    M -->|SSH TCP 22 using Ansible key| N1[Node 1 EC2<br/>Ubuntu]
    M -->|SSH TCP 22 using Ansible key| N2[Node 2 EC2<br/>Ubuntu]
    M -->|ansible / ansible-playbook| N1
    M -->|ansible / ansible-playbook| N2
    N1 --> D1[Docker + Devboard containers]
    N2 --> D2[Docker + Devboard containers]
    U[Browser] -->|HTTP TCP 8080, if app is running| N1
    U -->|HTTP TCP 8080, if app is running| N2
```

### SSH key flow

``` mermaid
flowchart LR
    W[Windows PC] -->|SSH key for master login| M[Ansible Master]
    M -->|Ansible private key stored securely on master| N1[Node 1]
    M -->|Same key only if its matching public key is authorized| N2[Node 2]
    M -.->|Public key only| A1[Node 1 ~/.ssh/authorized_keys]
    M -.->|Public key only| A2[Node 2 ~/.ssh/authorized_keys]
```

**Important:** SSH has two different key relationships: 1. Your Windows
PC uses a key authorized on the **master**. 2. The master uses an SSH
private key to connect to **node-1 and node-2**. The matching public key
must be in each node's `/home/ubuntu/.ssh/authorized_keys`.

Never copy a private key into `authorized_keys`; that file contains
**public keys only**.

## 3. Launch the EC2 instances

Create three Ubuntu EC2 instances in the same VPC, preferably the same
region:

  ----------------------------------------------------------------------------------
  Instance                 Purpose                 Example size            Root disk
  ------------------------ --------------- -------------------- --------------------
  `ansible-udaan-master`   Ansible control     `t3.micro` for a               20 GiB
                           node                       small lab 

  `node-1`                 Managed node        `t3.micro` for a               20 GiB
                                                      small lab 

  `node-2`                 Managed node        `t3.micro` for a               20 GiB
                                                      small lab 
  ----------------------------------------------------------------------------------

A `t3.micro` has limited memory; running multiple Docker containers and
a database may require a larger instance. Monitor memory and CPU.

### Security group rules

Prefer a security group for the master and one for the managed nodes.

**Master security group** - Inbound SSH TCP 22: **your current public IP
only**. - Outbound SSH TCP 22 to the managed-node security group (or VPC
range). - Outbound HTTPS TCP 443 for package and Git downloads. - HTTP
TCP 80 is not required for Ansible itself. Add it only if you host a web
service on the master.

**Managed-node security group** - Inbound SSH TCP 22: **source = the
master's security group**, not `0.0.0.0/0`. - Inbound application TCP
8080 (and any other required app ports): restrict to your IP or load
balancer/security group. Avoid exposing database port 5432 publicly. -
Outbound HTTPS TCP 443 and other required package/application traffic.

Use the AWS console's security-group references where possible. Do not
open SSH to the entire internet for convenience.

## 4. Connect from Windows to the master

In PowerShell, move to the directory containing the downloaded master
key:

``` powershell
cd $HOME\Downloads
ssh -i .\ansible-master-login.pem ubuntu@MASTER_PUBLIC_DNS
```

Use the actual key filename and master public DNS/IP from the EC2
console. If SSH reports an unprotected private key on Linux/macOS, fix
permissions with `chmod 400 key.pem`. Windows ACL permissions may need
to be restricted through file Properties → Security.

## 5. Prepare the Ansible master

Connect to the master, then run:

``` bash
sudo apt update
sudo apt install -y ansible openssh-client git
ansible --version
```

Create directories:

``` bash
mkdir -p ~/keys ~/playbooks
chmod 700 ~/keys
```

### Put the managed-node private key on the master safely

Use a dedicated key for automation. Prefer AWS Systems Manager/Secrets
Manager or another secure secret-delivery process. For a small lab, if
you securely transfer a downloaded key to the master, do not paste its
contents into chat or a README.

After the key is securely placed at `~/keys/ansible-nodes.pem`:

``` bash
chmod 400 ~/keys/ansible-nodes.pem
ls -l ~/keys/ansible-nodes.pem
```

The key should be readable only by its owner. Do not run `cat` on the
private key to display it.

## 6. Configure SSH access from master to both nodes

Generate a dedicated key on the master **only if you intend to use a new
key pair**:

``` bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_nodes_ed25519
chmod 600 ~/.ssh/ansible_nodes_ed25519
chmod 644 ~/.ssh/ansible_nodes_ed25519.pub
```

Add the contents of `~/.ssh/ansible_nodes_ed25519.pub` to
`/home/ubuntu/.ssh/authorized_keys` on **both** managed nodes using a
trusted access method. The file must contain the public key on one line.
On each node:

``` bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

If you instead use an AWS-downloaded PEM key, its corresponding public
key must already be authorized on each node. Do not assume the
master-login key also grants access to the managed nodes.

Test each node from the master:

``` bash
ssh -i ~/keys/ansible-nodes.pem ubuntu@NODE_1_PRIVATE_IP
ssh -i ~/keys/ansible-nodes.pem ubuntu@NODE_2_PRIVATE_IP
```

For nodes in the same VPC, prefer their **private IP addresses** for
master-to-node traffic. Exit the SSH session after each test.

## 7. Inventory file: `/etc/ansible/hosts`

Create or edit the default inventory:

``` bash
sudo mkdir -p /etc/ansible
sudo nano /etc/ansible/hosts
```

Use this inventory template, replacing the example private IPs and key
path:

``` ini
[servers]
node-1 ansible_host=172.31.14.183
node-2 ansible_host=172.31.13.223

[servers:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/home/ubuntu/keys/ansible-nodes.pem
```

Use each instance's actual **private IPv4 address** from the EC2
console. The IPs above are examples from the lab notes and may have
changed. Make sure the key path exactly matches the file on the master.

Check that Ansible can parse the inventory:

``` bash
ansible-inventory --list
ansible-inventory --graph
```

> Inventory syntax must be `[servers:vars]` with no space before `vars`.
> The group name must match the host group, such as `servers`.

## 8. Test connectivity and run ad-hoc commands

``` bash
# Ping both managed nodes
ansible servers -m ping

# Show uptime
ansible servers -a "uptime"

# Gather facts
ansible servers -m setup

# Run a command as root (sudo)
ansible servers -b -m command -a "apt-get update"
```

Expected ping output includes `SUCCESS` and `"ping": "pong"` for each
node.

If you use a non-default inventory file instead, add
`-i /path/to/hosts.ini` to each command.

## 9. Playbook: install common packages and Docker

Create the playbook:

``` bash
nano ~/playbooks/install_packages.yml
```

Paste this valid YAML:

``` yaml
---
- name: Install common packages on all managed nodes
  hosts: servers
  become: true

  tasks:
    - name: Refresh apt package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install unzip and Git
      ansible.builtin.apt:
        name:
          - unzip
          - git
        state: present

    - name: Install Docker Engine package
      ansible.builtin.apt:
        name: docker.io
        state: present

    - name: Install Docker Compose plugin
      ansible.builtin.apt:
        name: docker-compose-v2
        state: present

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Add ubuntu user to docker group
      ansible.builtin.user:
        name: ubuntu
        groups: docker
        append: true
```

Validate and execute:

``` bash
ansible-playbook --syntax-check ~/playbooks/install_packages.yml
ansible-playbook ~/playbooks/install_packages.yml
```

Group membership changes generally apply to new login sessions. Log out
and reconnect before running Docker without `sudo`, or use
`sudo docker ...`. Membership in the `docker` group grants
root-equivalent privileges.

Verify on both nodes:

``` bash
ansible servers -b -m command -a "docker --version"
ansible servers -b -m command -a "docker compose version"
ansible servers -b -m command -a "docker ps"
```

## 10. Playbook: clone and run Devboard with Docker Compose

First confirm that the repository's `docker-compose.yml` is configured
for the intended ports, environment variables, and image/build strategy.
The example below assumes the repository is public and has a usable
`.env.example`.

Create:

``` bash
nano ~/playbooks/run_devboard.yml
```

``` yaml
---
- name: Deploy Devboard on managed nodes
  hosts: servers
  become: true

  tasks:
    - name: Clone or update Devboard repository
      ansible.builtin.git:
        repo: "https://github.com/anandkumarprajap/GitHub-Actions-Revision.git"
        dest: /home/ubuntu/devboard-app
        version: master
        update: true
      become_user: ubuntu

    - name: Copy example environment file if .env is absent
      ansible.builtin.copy:
        src: /home/ubuntu/devboard-app/.env.example
        dest: /home/ubuntu/devboard-app/.env
        remote_src: true
        force: false
        owner: ubuntu
        group: ubuntu
        mode: "0600"

    - name: Start Devboard with Docker Compose
      ansible.builtin.command:
        argv:
          - docker
          - compose
          - up
          - -d
        chdir: /home/ubuntu/devboard-app
```

Run it:

``` bash
ansible-playbook --syntax-check ~/playbooks/run_devboard.yml
ansible-playbook ~/playbooks/run_devboard.yml
```

**Important deployment notes** - The example `.env` file may contain
placeholder values. Set correct database credentials and application
settings securely before deploying. - If the Compose file builds images
locally, the required Dockerfiles and build context must be present. If
it pulls images, confirm the image names and tags exist. - Deploying the
same app to two nodes creates two separate application stacks and may
create separate databases unless you configure shared external services.
This is not automatically load-balanced or highly available. - Do not
expose PostgreSQL port 5432 to the internet. - If the app uses ports
8080 and 8081, allow only the ports actually needed in the security
group.

Verify:

``` bash
ansible servers -a "docker ps"
ansible servers -a "docker compose version"
```

Test the application in a browser using `http://NODE_PUBLIC_IP:8080`
only if the app maps container port 4173 to host port 8080 and the
security group permits your IP. Prefer a load balancer for a real
deployment.

## 11. Useful Ansible command shortcuts

  --------------------------------------------------------------------------------------------------------
  Purpose                             Command
  ----------------------------------- --------------------------------------------------------------------
  Check inventory hosts               `ansible servers --list-hosts`

  Ping all nodes                      `ansible servers -m ping`

  Run uptime                          `ansible servers -a "uptime"`

  Run as sudo/root                    `ansible servers -b -m command -a "whoami"`

  Check Docker                        `ansible servers -b -m command -a "docker --version"`

  Syntax-check a playbook             `ansible-playbook --syntax-check ~/playbooks/install_packages.yml`

  Run package playbook                `ansible-playbook ~/playbooks/install_packages.yml`

  Run app playbook                    `ansible-playbook ~/playbooks/run_devboard.yml`

  Limit to node 1                     `ansible-playbook ~/playbooks/install_packages.yml --limit node-1`

  Limit to node 2                     `ansible-playbook ~/playbooks/install_packages.yml --limit node-2`

  Verbose troubleshooting             `ansible servers -m ping -vvv`
  --------------------------------------------------------------------------------------------------------

Because the inventory is at `/etc/ansible/hosts`, Ansible automatically
uses it by default. If you use `~/inventories/hosts.ini`, specify
`-i ~/inventories/hosts.ini`.

## 12. Troubleshooting

### `UNPROTECTED PRIVATE KEY FILE` / `bad permissions`

``` bash
chmod 400 ~/keys/ansible-nodes.pem
```

### `Permission denied (publickey)`

-   Confirm the username is `ubuntu` for Ubuntu EC2 images.
-   Confirm the private key matches the public key in the node's
    `~/.ssh/authorized_keys`.
-   Confirm the inventory key path is correct.
-   Test SSH manually from the master before running Ansible.

### `Host key verification failed`

Do not routinely delete all of `~/.ssh/known_hosts`. Verify the
instance's identity first. If an instance was intentionally replaced and
its host key changed, remove only that host entry:

``` bash
ssh-keygen -R NODE_PRIVATE_IP
```

Then reconnect and verify the host fingerprint through a trusted
channel.

### `docker: permission denied ... docker.sock`

Use `sudo docker ps` to verify Docker first. Then reconnect after adding
`ubuntu` to the `docker` group, or keep using `sudo`. Do not change the
Docker socket to world-writable permissions.

### Inventory parse error

Check exact syntax:

``` ini
[servers]
node-1 ansible_host=172.31.14.183
node-2 ansible_host=172.31.13.223

[servers:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/home/ubuntu/keys/ansible-nodes.pem
```

### App is not reachable

-   Run `docker ps` and `docker compose logs --tail=100` in the app
    directory.
-   Check Compose port mappings and app health.
-   Check the EC2 security group, OS firewall, and whether the app binds
    to `0.0.0.0` inside its container.
-   Use the public IP only for browser access from outside the VPC; use
    private IPs for master-to-node SSH.

## 13. Final checklist

-   [ ] Three Ubuntu EC2 instances are running and pass status checks.
-   [ ] SSH to the master works from your Windows PC.
-   [ ] The master can SSH to both nodes using private IPs.
-   [ ] The inventory lists `node-1` and `node-2`.
-   [ ] `ansible servers -m ping` succeeds on both nodes.
-   [ ] The package playbook completes without failures.
-   [ ] Docker and Docker Compose work on both nodes.
-   [ ] Devboard environment values and port mappings are correct.
-   [ ] Security groups expose only necessary ports and sources.
-   [ ] Any private key pasted or shared previously has been
    rotated/replaced.
