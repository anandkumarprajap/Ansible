# Ansible for DevOps Engineers

## 1. What Is Ansible?

**Ansible** is an open-source IT automation tool used for configuration management, application deployment, and infrastructure automation.

It helps DevOps engineers automate repetitive tasks across multiple servers without configuring each server manually.

### Examples of Ansible Tasks

- Install Docker and Jenkins on EC2 instances.
- Configure Linux servers.
- Install software and dependencies.
- Start, stop, and restart services.
- Deploy applications.
- Manage multiple servers using a single playbook.

**In simple words:** Ansible allows us to write instructions once and execute them across multiple servers automatically.

---

## 2. Why Do We Need Ansible?

Imagine Rohan is a system administrator. He needs to install Docker on 100 EC2 instances.

### Without Ansible — Manual Work

1. Connect to the first EC2 instance using SSH.
2. Update the package repository.
3. Install Docker.
4. Start the Docker service.
5. Repeat the same steps on the remaining 99 servers.

This process takes time and increases the chances of human error.

### With Ansible — Automation

1. Rohan creates an Ansible playbook.
2. He defines all the target servers in an inventory file.
3. He runs the playbook once.
4. Ansible connects to the servers and performs the required tasks.

The same playbook can configure hundreds of servers. Actual execution time depends on the number of servers, network speed, and task complexity.

### Manual Work vs Ansible

| Manual Configuration | Ansible Automation |
|---|---|
| Repeated commands | Reusable playbooks |
| More manual effort | Less manual effort |
| Time-consuming at scale | Can configure many servers in parallel |
| Greater risk of inconsistent configuration | More consistent configuration |
| Difficult to repeat reliably | Easier to repeat and maintain |

### Benefits of Ansible

- Reduces manual work.
- Saves time on repetitive tasks.
- Improves configuration consistency.
- Automates software installation.
- Supports application deployment.
- Helps DevOps teams deliver applications faster.

---

## 3. Ansible Architecture

Ansible commonly uses SSH to communicate with Linux servers. A separate Ansible agent is generally not required on managed Linux hosts.

```text
              DevOps Engineer
                     |
                     v
          +----------------------+
          |    Control Node      |
          |----------------------|
          | Ansible              |
          | Inventory File       |
          | YAML Playbook        |
          +----------------------+
                     |
                SSH Connection
                     |
        +------------+------------+
        |            |            |
        v            v            v
   +---------+  +---------+  +---------+
   | EC2-1   |  | EC2-2   |  | EC2-100 |
   | Docker  |  | Docker  |  | Docker  |
   | Jenkins |  | Jenkins |  | Jenkins |
   +---------+  +---------+  +---------+
```

### Important Components

| Component | Description |
|---|---|
| Control Node | Machine where Ansible runs |
| Inventory | List of target servers |
| Managed Nodes | Servers that Ansible configures |
| SSH | Common connection method for Linux hosts |
| Playbook | YAML file containing automation instructions |
| Tasks | Individual operations to execute |
| Modules | Tools used to perform tasks such as package installation |

---

## 4. What Is YAML?

**YAML** stands for *YAML Ain't Markup Language*.

It is a human-readable format used to store structured data. Ansible uses YAML to define playbooks, while Kubernetes commonly uses YAML manifests to define resources.

### Example: Personal Information in YAML

Create a file named `person.yaml`.

```yaml
---
name: "Jan Kumar"
age: 25
living_in: "Ranchi"
from: "Bokaro"

hobbies:
  - teaching
  - fitness
  - management
  - public speaking
```

### Understanding YAML

- `name`, `age`, and `living_in` are keys.
- `"Jan Kumar"`, `25`, and `"Ranchi"` are values.
- `hobbies` is a list.
- A hyphen (`-`) represents a list item.
- Indentation defines the structure of the data.

**Important:** Use consistent indentation, commonly two spaces. Do not use tabs for YAML indentation.

### Ansible YAML vs Kubernetes YAML

| Ansible | Kubernetes |
|---|---|
| YAML playbook | YAML manifest |
| Automates configuration and tasks | Defines desired Kubernetes resources |
| Uses `hosts`, `tasks`, and modules | Uses fields such as `apiVersion`, `kind`, and `spec` |

---

## 5. Difference Between Terraform and Ansible

Terraform and Ansible are different tools, but they can work together in a DevOps workflow.

### Terraform — Infrastructure Provisioning

Terraform is primarily used to create and manage infrastructure resources.

Examples:

- Create an AWS VPC.
- Create public and private subnets.
- Create EC2 instances.
- Configure security groups.
- Provision an Amazon EKS cluster.

### Ansible — Configuration Management

Ansible is primarily used to configure systems and automate operational tasks.

Examples:

- Install Docker on EC2 instances.
- Configure Jenkins.
- Install application dependencies.
- Manage Linux services.
- Deploy applications to supported environments.

### Comparison Table

| Feature | Terraform | Ansible |
|---|---|---|
| Main purpose | Infrastructure provisioning | Configuration management and automation |
| Language | HCL | YAML playbooks |
| Example | Create EC2 instances | Install Docker on EC2 |
| Infrastructure | Creates and manages resources | Can also automate infrastructure through supported APIs |
| Typical workflow | Plan and apply | Run a playbook |

### How They Work Together

```text
       DevOps Engineer
              |
              v
        +-------------+
        |  Terraform  |
        |-------------|
        | Create VPC  |
        | Create EC2  |
        | Create EKS  |
        +-------------+
              |
              v
        +-------------+
        |   Ansible   |
        |-------------|
        | Configure   |
        | Install     |
        | Automate    |
        +-------------+
              |
              v
        +-------------+
        | Application |
        | Deployment  |
        +-------------+
```

**Remember:**

- Terraform: Create and manage infrastructure.
- Ansible: Configure systems and automate tasks.
- Docker: Package applications into containers.
- Kubernetes: Orchestrate containerized applications.

These tools have different responsibilities and can be combined in one workflow.

---

## 6. Practical Example: Install Docker Using Ansible

### Step 1: Install Ansible

On an Ubuntu control node:

```bash
sudo apt update
sudo apt install ansible -y

ansible --version
```

### Step 2: Create an Inventory File

Create `inventory.ini`.

```ini
[webservers]
server1 ansible_host=10.0.1.10
server2 ansible_host=10.0.1.11

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/my-key.pem
```

Replace the example IP addresses and private key path with your actual values.

### Step 3: Check Connectivity

```bash
ansible webservers -i inventory.ini -m ansible.builtin.ping
```

This checks whether Ansible can connect to the managed servers and execute its ping module.

### Step 4: Create an Ansible Playbook

Create `install_docker.yml`.

```yaml
---
- name: Install Docker on EC2 servers
  hosts: webservers
  become: true

  tasks:
    - name: Update apt package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Docker
      ansible.builtin.apt:
        name: docker.io
        state: present

    - name: Start and enable Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true
```

### Step 5: Execute the Playbook

```bash
ansible-playbook -i inventory.ini install_docker.yml
```

### Step 6: Verify Docker

```bash
ansible webservers -i inventory.ini \
  -a "docker --version"
```

**Result:** Ansible installs Docker and ensures that the Docker service is running on both target servers.

Prerequisites include SSH access, suitable privileges, network connectivity, and Python on the managed hosts. This example uses Ubuntu/Debian package management.

---

## 7. How Can I Learn Ansible?

Follow this learning roadmap from beginner to intermediate level.

### Level 1: Linux Fundamentals

Learn:

- Linux commands.
- File permissions and ownership.
- Users and groups.
- SSH connections.
- Package management.
- Services and processes.
- Basic networking.

**Practice:** Create and connect to Ubuntu EC2 instances.

### Level 2: YAML Fundamentals

Learn:

- Keys and values.
- Lists.
- Nested objects.
- Indentation.
- Boolean and numeric values.

**Practice:** Create `person.yaml` and understand its structure.

### Level 3: Ansible Basics

Learn:

- Installing Ansible.
- Inventory files.
- Ad-hoc commands.
- Modules.
- Playbooks.
- Tasks.
- Variables.
- Privilege escalation with `become`.

**Practice:** Install Nginx or Docker on two EC2 instances.

### Level 4: Intermediate Ansible

Learn:

- Loops.
- Conditions.
- Handlers.
- Templates.
- Roles.
- Ansible Vault.
- Error handling.
- Idempotency.

**Practice:** Create reusable playbooks for installing Docker and configuring Jenkins.

### Level 5: DevOps Projects

Combine Ansible with:

- AWS and EC2.
- Terraform.
- Docker.
- Kubernetes.
- Jenkins and GitHub Actions.

**Practice project:** Provision EC2 instances using Terraform, configure them with Ansible, build an application image with Docker, and deploy it to a suitable runtime.

---

## 8. Important Ansible Commands

| Command | Purpose |
|---|---|
| `ansible --version` | Check the Ansible version |
| `ansible-inventory -i inventory.ini --list` | Display inventory details |
| `ansible all -i inventory.ini -m ansible.builtin.ping` | Test connectivity to all inventory hosts |
| `ansible all -i inventory.ini -m ansible.builtin.setup` | Gather host facts |
| `ansible-playbook -i inventory.ini playbook.yml --syntax-check` | Check playbook syntax |
| `ansible-playbook -i inventory.ini playbook.yml --check` | Preview supported changes in check mode |
| `ansible-playbook -i inventory.ini playbook.yml` | Execute the playbook |

---

## 9. Important Interview Questions

**Q1. What is Ansible?**

Ansible is an open-source automation tool used for configuration management, application deployment, and IT task automation.

**Q2. Why do we need Ansible?**

To reduce repetitive manual work, automate configuration, and manage multiple servers consistently.

**Q3. What is an Ansible playbook?**

A YAML file that describes the plays and tasks Ansible executes on selected hosts.

**Q4. What is an inventory file?**

A file or other inventory source that defines the hosts and groups Ansible manages.

**Q5. Does Ansible require an agent?**

Ansible generally manages Linux hosts over SSH without a separate Ansible agent.

**Q6. What is the difference between Terraform and Ansible?**

Terraform primarily provisions and manages infrastructure resources. Ansible primarily configures systems and automates operational tasks.

**Q7. What is idempotency?**

Idempotency means that repeating an operation should leave the system in the intended state without unnecessary changes.

---

## 10. Official Learning Resources

- [Ansible Getting Started](https://docs.ansible.com/projects/ansible/latest/getting_started/)
- [Ansible Playbook Guide](https://docs.ansible.com/projects/ansible/latest/playbook_guide/index.html)
- [Ansible Documentation](https://docs.ansible.com/)
- [Red Hat Ansible Learning](https://developers.redhat.com/learn/ansible)

---

## Conclusion

Ansible is an important tool for DevOps engineers and system administrators because it automates repetitive configuration and deployment tasks.

Start with Linux and YAML, learn inventory files and playbooks, and practice installing software on multiple EC2 instances.

**Your first goal:** Write one Ansible playbook that installs Docker on two Ubuntu EC2 instances. Then extend it to more servers and learn reusable roles.

*Learn once, automate repeatedly, and reduce manual effort.*
