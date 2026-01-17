# Enterprise Multi-Server Infrastructure Automation Platform

Production-grade automation framework for managing distributed Linux server infrastructure with centralized orchestration, security compliance, and operational monitoring.

## Architecture Overview

This platform provides:

- Parallel server provisioning and configuration management via Ansible
- Automated SSH key distribution and passwordless authentication
- Encrypted database backups (MySQL/PostgreSQL) with S3 integration
- Security auditing and compliance validation
- Centralized health monitoring and HTML dashboard generation

**Deployment Model:** Push-based automation from central control node to 10+ managed servers across staging and production environments.

## Prerequisites

### Control Node Requirements

- Linux system (RHEL 8+/Ubuntu 20.04+)
- Python 3.8+
- Ansible 2.12+
- AWS CLI configured with S3 access
- SSH access to all managed nodes

### Managed Node Requirements

- Linux OS (RHEL/CentOS 7+, Ubuntu 18.04+)
- Python 3.6+
- sudo access for automation user

### AWS Resources

- S3 bucket for backup storage
- IAM user with S3 PutObject/GetObject permissions

## Initial Setup

### 1. Control Node Configuration

```bash
# Install dependencies
sudo yum install -y python3 python3-pip ansible git
pip3 install boto3 jinja2 paramiko

# Clone repository
git clone <repository-url>
cd infra-automation-platform

# Configure AWS credentials
aws configure
# Enter Access Key, Secret Key, Region (us-east-1), Output (json)

# Generate SSH key for automation
ssh-keygen -t ed25519 -f ~/.ssh/automation_key -N ""
```

### 2. Inventory Configuration

Edit `ansible/inventory/production/hosts.yml`:

```yaml
all:
  children:
    web_servers:
      hosts:
        web01: { ansible_host: 10.0.1.10 }
        web02: { ansible_host: 10.0.1.11 }
    database_servers:
      hosts:
        db01: { ansible_host: 10.0.2.10 }
        db02: { ansible_host: 10.0.2.11 }
```

Update group variables in `ansible/inventory/production/group_vars/`:

- `all.yml`: Global settings (SSH user, key path, S3 bucket)
- `database_servers.yml`: Database-specific config
- `web_servers.yml`: Web server-specific config

### 3. SSH Key Distribution

```bash
# Copy public key to ansible/roles/ssh-keys/files/
cp ~/.ssh/automation_key.pub ansible/roles/ssh-keys/files/

# Distribute keys (requires initial password authentication)
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/provision.yml \
  --tags ssh-keys \
  --ask-pass
```

### 4. Full Provisioning

```bash
# Complete server provisioning
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/provision.yml
```

## Operational Usage

### Deploy Applications

```bash
# Deploy to all servers
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/deploy.yml

# Deploy to specific group
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/deploy.yml \
  --limit web_servers
```

### Database Backups

```bash
# Manual backup execution
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/database-backup.yml

# Automated backups run via systemd timers (daily at 02:00 UTC)
# Check status: systemctl status backup.timer
```

### Security Audits

```bash
# Run compliance scan
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/security-audit.yml

# View results in /var/log/security-audit/
```

### Health Monitoring

```bash
# Collect metrics and generate dashboard
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/health-check.yml

# Dashboard available at: /var/www/html/dashboard/index.html
```

### Parallel Execution

```bash
# Deploy across all servers in parallel (10 forks)
./scripts/orchestration/parallel-deploy.sh production

# Rolling update (zero-downtime)
./scripts/orchestration/rolling-update.sh production web_servers
```

## Common Failure Points

### SSH Authentication Failures

**Symptom:** "Permission denied (publickey)"
**Resolution:**

```bash
# Verify key is in authorized_keys on target
ssh -i ~/.ssh/automation_key user@target-host cat ~/.ssh/authorized_keys

# Re-run SSH key distribution
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/provision.yml \
  --tags ssh-keys \
  --ask-pass
```

### Backup Upload to S3 Fails

**Symptom:** boto3.exceptions.NoCredentialsError
**Resolution:**

```bash
# Verify AWS credentials
aws s3 ls s3://your-backup-bucket/

# Check IAM permissions include s3:PutObject
# Ensure scripts/backup/backup-upload.py uses correct bucket name
```

### Ansible Playbook Hangs

**Symptom:** Task execution timeout
**Resolution:**

```bash
# Increase timeout in ansible.cfg
# [defaults]
# timeout = 60

# Check network connectivity
ansible all -i ansible/inventory/production/hosts.yml -m ping

# Run with verbose logging
ansible-playbook -vvv ...
```

### Database Backup Corruption

**Symptom:** Restore fails with "corrupted archive"
**Resolution:**

```bash
# Verify backup integrity
gpg --decrypt /backup/mysql_YYYYMMDD.sql.gz.gpg | gunzip -t

# Check disk space on backup target
df -h /backup

# Review backup script logs
journalctl -u backup.service -n 50
```

## Maintenance

### Update Ansible Roles

```bash
git pull origin main
ansible-playbook -i ansible/inventory/production/hosts.yml \
  ansible/playbooks/provision.yml \
  --tags common,security
```

### Rotate SSH Keys

```bash
# Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/automation_key_new -N ""

# Update ansible/roles/ssh-keys/files/ with new public key
# Run provision playbook with ssh-keys tag
# Remove old key from authorized_keys after verification
```

### Backup Retention Management

```bash
# S3 lifecycle policy auto-deletes backups older than 90 days
# Adjust in configs/aws/s3-lifecycle.json if needed
```

## Performance Metrics

- **Provisioning Time:** 6 minutes per server (parallel execution across 10 servers)
- **Backup Completion:** MySQL (5GB) - 8 minutes, PostgreSQL (3GB) - 5 minutes
- **Security Audit:** 90 seconds per server
- **Dashboard Generation:** 15 seconds (aggregating 10 servers)

## Security Considerations

- SSH keys protected with filesystem permissions (0600)
- Database backups encrypted with GPG (4096-bit RSA)
- S3 transfer uses TLS 1.2+
- Fail2ban configured for SSH brute-force protection
- SELinux enforcing mode on RHEL systems
- Audit logs centralized in /var/log/audit/
