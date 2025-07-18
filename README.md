# Ansible Automation Platform 2.5 Operator Deployment on OpenShift

This project provides a comprehensive Ansible automation framework for deploying and managing Red Hat Ansible Automation Platform (AAP) 2.5 using operators on OpenShift Container Platform, with integrated Event Driven Ansible (EDA) capabilities.

## Architecture Overview

This deployment utilizes the AAP 2.5 operator to deploy the following components on OpenShift:

- **AAP Controller** - Web-based UI and API for managing Ansible automation
- **AAP Hub** - Private Galaxy server for hosting and managing Ansible collections
- **Event Driven Ansible (EDA)** - Event-driven automation and response system
- **PostgreSQL** - Database backend (operator-managed)
- **Redis** - Caching and message broker (operator-managed)

## Prerequisites

### OpenShift Cluster Requirements

- OpenShift Container Platform 4.12+ 
- Minimum 8 CPU cores and 32GB RAM across cluster nodes
- Storage class configured (e.g., `gp3-csi`, `standard`)
- Cluster admin privileges
- Red Hat operators catalog available

### Local Environment

- Ansible 2.14+
- Python 3.8+
- Required Python packages:
  ```bash
  pip install kubernetes openshift pyyaml
  ```
- OpenShift CLI (`oc`) configured and authenticated
- Valid AAP license file
- Red Hat Registry credentials

## Project Structure

```
aap-eda-management/
├── ansible.cfg                          # Ansible configuration
├── requirements.yml                     # Ansible collections and roles
├── inventories/                         # Environment-specific inventories
│   ├── production/hosts.yml            # Production OpenShift cluster
│   └── development/hosts.yml           # Development OpenShift cluster
├── group_vars/                         # Global variables
│   ├── all.yml                        # Common configuration
│   └── vault.yml                      # Encrypted sensitive data
├── playbooks/                          # Ansible playbooks
│   ├── site.yml                       # Main orchestration playbook
│   ├── openshift-operator-install.yml # AAP operator installation
│   ├── controller.yml                 # AAP Controller deployment
│   ├── hub.yml                        # AAP Hub deployment
│   ├── eda.yml                        # EDA Controller deployment
│   └── tasks/                         # Reusable task files
│       └── validate_prerequisites.yml # Deployment validation
└── rulebooks/                          # EDA rulebooks
    └── aap-monitoring.yml              # AAP monitoring and alerting
```

## Quick Start

### 1. Clone and Setup

```bash
git clone <repository-url>
cd aap-eda-management
```

### 2. Install Dependencies

```bash
ansible-galaxy install -r requirements.yml
```

### 3. Configure Environment

Edit the inventory file for your environment:

```bash
# For production
vi inventories/production/hosts.yml

# For development
vi inventories/development/hosts.yml
```

Update the OpenShift cluster details:
- `ansible_host`: OpenShift API server URL
- `k8s_auth_kubeconfig`: Path to your kubeconfig file
- `openshift_domain`: Your OpenShift apps domain

### 4. Configure Secrets

```bash
# Edit and encrypt the vault file
vi group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml
```

Required secrets:
- `vault_aap_license_file`: Base64-encoded AAP license
- `vault_redhat_registry_secret`: Red Hat registry pull secret
- `vault_aap_admin_password`: AAP admin password
- Database and Redis passwords

### 5. Validate Prerequisites

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --extra-vars "phase=validate" --ask-vault-pass
```

### 6. Deploy AAP

Full deployment:
```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --ask-vault-pass
```

Phased deployment:
```bash
# Install operator only
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --extra-vars "phase=operator" --ask-vault-pass

# Deploy Controller only
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --extra-vars "phase=controller" --ask-vault-pass

# Deploy Hub only
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --extra-vars "phase=hub" --ask-vault-pass

# Deploy EDA only
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --extra-vars "phase=eda" --ask-vault-pass
```

## Configuration

### Key Variables

Located in `group_vars/all.yml`:

```yaml
# AAP Configuration
aap_version: "2.5"
aap_namespace: "ansible-automation-platform"
aap_operator_channel: "stable-2.5-cluster-scoped"

# Resource Configuration
controller_replicas: 2
hub_replicas: 1
eda_replicas: 2

# Storage Configuration
storage_class: "gp3-csi"
controller_storage_size: "20Gi"
hub_storage_size: "100Gi"
postgres_storage_size: "50Gi"

# Network Configuration
controller_hostname: "controller.apps.openshift.example.com"
hub_hostname: "hub.apps.openshift.example.com"
eda_hostname: "eda.apps.openshift.example.com"
```

### Environment-Specific Settings

Production (`inventories/production/hosts.yml`):
- High availability with multiple replicas
- Larger resource allocations
- Production-grade storage classes
- SSL certificates via cert-manager

Development (`inventories/development/hosts.yml`):
- Single replicas for components
- Smaller resource allocations
- Basic storage classes
- Self-signed certificates

## Event Driven Ansible

The project includes EDA rulebooks for automated operations:

### AAP Monitoring Rulebook (`rulebooks/aap-monitoring.yml`)

Monitors and responds to:
- Pod crashes and restarts
- Storage usage alerts
- High CPU/memory usage
- Database connectivity issues
- SSL certificate expiration
- Backup failures

### Event Sources

- **Prometheus/Alertmanager**: OpenShift monitoring alerts
- **Webhooks**: AAP job notifications
- **Kubernetes Events**: OpenShift cluster events

## Accessing AAP Components

After deployment, access the components via their OpenShift routes:

```bash
# Get route URLs
oc get routes -n ansible-automation-platform

# Controller
https://controller.apps.openshift.example.com

# Hub
https://hub.apps.openshift.example.com

# EDA Controller
https://eda.apps.openshift.example.com
```

Default credentials:
- Username: `admin`
- Password: Check the `aap-admin-password` secret

## Monitoring and Maintenance

### Health Checks

```bash
# Check operator status
oc get csv -n ansible-automation-platform

# Check component status
oc get automationcontroller,automationhub,edacontroller -n ansible-automation-platform

# Check pod status
oc get pods -n ansible-automation-platform
```

### Scaling

```bash
# Scale Controller replicas
oc patch automationcontroller controller -n ansible-automation-platform \
  --type='merge' -p='{"spec":{"replicas":3}}'

# Scale EDA replicas
oc patch edacontroller eda-controller -n ansible-automation-platform \
  --type='merge' -p='{"spec":{"replicas":3}}'
```

### Backup and Recovery

The deployment includes automated backup configuration:
- Daily backups via CronJobs
- Retention policy: 30 days
- Backup storage on persistent volumes

## Troubleshooting

### Common Issues

1. **Operator Installation Fails**
   ```bash
   # Check operator logs
   oc logs -f deployment/ansible-automation-platform-operator-controller-manager \
     -n ansible-automation-platform
   ```

2. **Component Deployment Stuck**
   ```bash
   # Check custom resource status
   oc describe automationcontroller controller -n ansible-automation-platform
   ```

3. **Database Connection Issues**
   ```bash
   # Check PostgreSQL pod logs
   oc logs -f deployment/controller-postgres-13 -n ansible-automation-platform
   ```

### Support

For issues specific to this deployment framework:
1. Check the Ansible playbook logs
2. Verify OpenShift cluster resources
3. Validate configuration variables
4. Review EDA rulebook logs

For AAP-specific issues, consult the [Red Hat AAP documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform).

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## Security

- All sensitive data is encrypted using Ansible Vault
- Network policies restrict pod-to-pod communication
- RBAC is enforced for all components
- SSL/TLS encryption is enabled by default
