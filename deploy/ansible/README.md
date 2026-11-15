# Jenkins Automation with Ansible

This directory contains an automated Ansible playbook to provision and configure **Jenkins LTS** on Amazon Linux 2023 EC2, including scoped sudoers rules and Nginx reverse proxy configuration.

---

## Dynamic Domain Configuration

The domain is **fully dynamic** and can be supplied at runtime using any of these methods:

### Method 1: CLI Extra Vars (Recommended)
Pass `-e "jenkins_domain=..."` when running the playbook:
```bash
ansible-playbook -i "localhost," -c local setup-jenkins.yml -e "jenkins_domain=jenkins.yourcustomdomain.com"
```

### Method 2: Environment Variable
Export `JENKINS_DOMAIN` before running:
```bash
export JENKINS_DOMAIN="jenkins.yourcustomdomain.com"
ansible-playbook -i "localhost," -c local setup-jenkins.yml
```

### Method 3: Default Fallback
If omitted, the domain defaults to `jenkins.ajuna.website` (defined in `vars/main.yml`).

---

## Directory Structure

```text
deploy/ansible/
├── ansible.cfg              # Ansible configuration (become sudo, disable host check)
├── inventory.ini            # Target hosts inventory
├── setup-jenkins.yml        # Main playbook
├── vars/
│   └── main.yml             # Configurable variables & dynamic domain lookup
├── templates/
│   ├── jenkins.conf.j2      # Nginx reverse proxy template (uses {{ jenkins_domain }})
│   └── jenkins_sudoers.j2   # Scoped sudoers template
└── README.md                # Usage guide
```

---

## How to Run

### Option A: Running directly on the EC2 instance (Localhost)
```bash
cd /home/ec2-user/ONYX/deploy/ansible
ansible-playbook -i "localhost," -c local setup-jenkins.yml -e "jenkins_domain=jenkins.ajuna.website"
```

### Option B: Running from your local computer
Update `inventory.ini` with your EC2 public IP and SSH key, then run:
```bash
ansible-playbook -i inventory.ini setup-jenkins.yml -e "jenkins_domain=jenkins.ajuna.website"
```

---

## Post-Run Steps

1. The playbook output prints your **Initial Admin Password**.
2. Run Certbot to enable HTTPS:
   ```bash
   sudo certbot --nginx -d <your-dynamic-domain>
   ```
3. Open `https://<your-dynamic-domain>` in your browser to complete Jenkins web setup.
