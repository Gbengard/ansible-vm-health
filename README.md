# Ansible VM Health Monitoring Project

![Ansible VM Health](images/Ansible-VM-Health.png)

This Ansible project monitors the health of AWS EC2 instances (VMs) by collecting CPU, memory, and disk usage metrics. It then generates a visually appealing, animated HTML report and emails it to a specified recipient.

The project targets running EC2 instances tagged with `Environment=dev`, dynamically discovers them using the AWS EC2 inventory plugin, gathers metrics, and sends a consolidated report via Gmail SMTP.


## Prerequisites

- An AWS account with:
  - One EC2 instance to act as the **Ansible control node (master)**:
    - Ubuntu-based AMI
    - Security group allowing inbound SSH (port 22) and SMTP (port 587)
  - At least one (preferably multiple) target EC2 instances:
    - Ubuntu-based AMI
    - Tagged with `Environment=dev`
    - Security group allowing inbound SSH (port 22) from the control node
    - Instance state: running
- A Gmail account with an **App Password** for SMTP (generated at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords))

## Setup Instructions

### 1. Launch EC2 Instances
- Launch 1 control node instance (Ansible master).
- Launch your target instances (e.g., 10) and tag them:
  - Key: `Environment`, Value: `dev`
  - (Optional) The provided tagging script can automatically name them `web-01`, `web-02`, etc.

### 2. Connect to the Control Node
SSH into the Ansible master instance.

### 3. Update System and Install Ansible
```bash
sudo apt update && sudo apt upgrade -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

### 4. Install and Configure AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure   # Provide your AWS Access Key, Secret Key, region (us-east-1), and output format
```

### 5. (Optional) Auto-Tag and Name Instances
Save the following as `tag-instances.sh`, make it executable, and run it:
```bash
#!/bin/bash
instance_ids=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=dev" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

sorted_ids=($(echo "$instance_ids" | tr '\t' '\n' | sort))

counter=1
for id in "${sorted_ids[@]}"; do
  name="web-$(printf "%02d" $counter)"
  echo "Tagging $id as $name"
  aws ec2 create-tags --resources "$id" --tags Key=Name,Value="$name"
  ((counter++))
done
```

### 6. Generate SSH Key Pair
```bash
ssh-keygen -t rsa -b 4096 -C "Ansible Master"   # Accept defaults or specify path
```

### 7. Project Structure and Configuration Files

Create the following directory structure and files:

```
.
├── ansible.cfg
├── inventory/
│   └── aws_ec2.yaml
├── group_vars/
│   └── all.yaml
├── templates/
│   └── report_email_animated.html.j2
├── collect_metrics.yaml
├── send_report.yaml
└── playbook.yaml
```

#### `ansible.cfg`
```ini
[defaults]
inventory = ./inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

#### `inventory/aws_ec2.yaml`
```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.Environment
    prefix: env
```

#### `group_vars/all.yaml`
```yaml
smtp_server: "smtp.gmail.com"
smtp_port: 587
email_user: "your-gmail@gmail.com"
email_pass: "your-app-password"
alert_recipient: "recipient@example.com"
```

#### Virtual Environment and Required Collections
```bash
sudo apt install python3-venv -y
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install boto3 botocore docker
ansible-galaxy collection install amazon.aws
```

Verify inventory:
```bash
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

### 8. Distribute SSH Public Key to Target Hosts
Save this script as `distribute-key.sh` (adjust `PEM_FILE` if needed):
```bash
#!/bin/bash
PEM_FILE="gbengard.pem"   # Your private key for initial SSH access
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"
INVENTORY_FILE="inventory/aws_ec2.yaml"

HOSTS=$(ansible-inventory -i $INVENTORY_FILE --list | jq -r '._meta.hostvars | keys[]')

for HOST in $HOSTS; do
  echo "Injecting key into $HOST"
  ssh -o StrictHostKeyChecking=no -i $PEM_FILE $USER@$HOST "
    mkdir -p ~/.ssh &&
    echo \"$PUB_KEY\" >> ~/.ssh/authorized_keys &&
    chmod 700 ~/.ssh &&
    chmod 600 ~/.ssh/authorized_keys
  "
done
```

Run the script to enable passwordless SSH from the control node.

### 9. Playbooks

#### `collect_metrics.yaml`
```yaml
- name: Collect VM metrics
  hosts: env_dev
  become: true
  gather_facts: true
  tasks:
    - name: Install sysstat (Debian/Ubuntu)
      apt:
        name: sysstat
        state: present
      when: ansible_os_family == "Debian"

    - name: Get CPU usage
      shell: "mpstat 1 1 | awk '/Average/ && $NF ~ /[0-9.]+/ {print 100 - $NF}'"
      register: cpu_usage

    - name: Get memory usage
      shell: "free | awk '/Mem/{printf(\"%.2f\", $3/$2 * 100.0)}'"
      register: mem_usage

    - name: Get disk usage (root)
      shell: "df / | awk 'NR==2 {print $5}' | tr -d '%'"
      register: disk_usage

    - name: Set metrics fact
      set_fact:
        vm_metrics:
          hostname: "{{ inventory_hostname }}"
          cpu: "{{ cpu_usage.stdout | float | round(2) }}"
          mem: "{{ mem_usage.stdout | float | round(2) }}"
          disk: "{{ disk_usage.stdout | float | round(2) }}"
```

#### `send_report.yaml`
```yaml
- name: Send consolidated VM report
  hosts: localhost
  gather_facts: true
  vars:
    collected_metrics: >-
      {{
        hostvars
        | dict2items
        | selectattr('value.vm_metrics', 'defined')
        | map(attribute='value.vm_metrics')
        | list
      }}
    timestamp: "{{ ansible_date_time.date }} {{ ansible_date_time.time }}"
    subject_line: "📊 VM Report – {{ ansible_date_time.date }} {{ ansible_date_time.hour }}:{{ ansible_date_time.minute }}"
  tasks:
    - name: Send HTML report via email
      community.general.mail:
        host: "{{ smtp_server }}"
        port: "{{ smtp_port }}"
        username: "{{ email_user }}"
        password: "{{ email_pass }}"
        to: "{{ alert_recipient }}"
        subject: "{{ subject_line }}"
        body: "{{ lookup('template', 'templates/report_email_animated.html.j2') }}"
        subtype: html
```

#### `playbook.yaml`
```yaml
- import_playbook: collect_metrics.yaml
- import_playbook: send_report.yaml
```

#### `templates/report_email_animated.html.j2`
(Full HTML template – identical to the one provided in the query. Paste the entire HTML code here in the repository.)

### 10. Run the Monitoring Playbook
```bash
ansible-playbook playbook.yaml
```

The playbook will:
1. Collect metrics from all dev instances.
2. Generate a styled HTML report with progress bars and health badges.
3. Email the report to the configured recipient.

![VM Reports](images/VM-Reports.png)


Schedule this playbook with cron for periodic monitoring:
```bash
crontab -e
# Example: every hour
0 * * * * cd /path/to/project && /path/to/ansible-env/bin/ansible-playbook playbook.yaml
```

Enjoy automated VM health insights! 🚀
