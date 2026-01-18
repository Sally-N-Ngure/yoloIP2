# YOLO E-commerce Application - Ansible Automated Deployment

## Overview

This project automates the deployment of a full-stack MERN (MongoDB, Express, React, Node.js) e-commerce application using **Ansible** for configuration management and **Vagrant** for VM provisioning. The application is containerized using Docker and deployed on an Ubuntu 20.04 VM.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Usage](#usage)
- [Ansible Roles](#ansible-roles)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Documentation](#documentation)

---

## Prerequisites

Before you begin, ensure you have the following installed on your host machine:

- **VirtualBox** (>= 6.1): [Download](https://www.virtualbox.org/wiki/Downloads)
- **Vagrant** (>= 2.2): [Download](https://www.vagrantup.com/downloads)
- **Ansible** (>= 2.9): 
  ```bash
  # On Ubuntu/Debian
  sudo apt update
  sudo apt install ansible
  
  # On macOS
  brew install ansible
  ```

### Verify Installations

```bash
# Check VirtualBox
vboxmanage --version

# Check Vagrant
vagrant --version

# Check Ansible
ansible --version
```

---

## Quick Start

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd yoloIP2
```

### 2. Start the Deployment

```bash
# Start VM and run Ansible provisioning
vagrant up
```

When prompted for the Vault password, enter: **12345**

### 3. Access the Application

Once deployment completes (5-10 minutes), access the application:

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:5000
- **MongoDB**: localhost:27017

### 4. Test the Application

1. Open http://localhost:3000 in your browser
2. Navigate to "Add Product"
3. Fill in product details and submit
4. Verify the product appears in the list
5. Refresh the page to confirm data persistence

---

## Project Structure

```
yoloIP2/
├── Vagrantfile                 # VM configuration
├── ansible.cfg                 # Ansible settings
├── hosts                       # Ansible inventory
├── playbook.yaml              # Main Ansible playbook
│
├── ansible/
│   ├── group_vars/
│   │   └── all.yml            # Global variables
│   ├── secrets.yml            # Encrypted secrets (Vault)
│   └── roles/
│       ├── install_docker/    # Docker installation
│       ├── install_git/       # Git installation
│       ├── clone_repo/        # Repository cloning
│       ├── network_deployment/# Docker network setup
│       ├── backend_deployment/# Backend + MongoDB
│       └── frontend_deployment/# Frontend React app
│
├── backend/                   # Node.js backend source
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
│
├── client/                    # React frontend source
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│
├── EXPLANATION.md             # Docker/Compose explanation (IP2)
├── ANSIBLE_EXPLANATION.md     # Ansible deployment explanation (IP3)
└── README.md                  # This file
```

---

## Configuration

### Vagrantfile

Configures the Ubuntu 20.04 VM with:
- Box: `geerlingguy/ubuntu2004` version 1.0.4
- Memory: 2048 MB
- CPUs: 2
- Port Forwarding:
  - 3000 (frontend)
  - 5000 (backend)
  - 27017 (MongoDB)

### Ansible Configuration (ansible.cfg)

```ini
[defaults]
inventory = hosts
remote_user = vagrant
private_key_file = .vagrant/machines/default/virtualbox/private_key
roles_path = ./ansible/roles
host_key_checking = False
```

### Variables (ansible/group_vars/all.yml)

Key configuration variables:
```yaml
app_directory: /home/vagrant/yolo
repo_url: https://github.com/douglaswangome/yolo.git
docker_network_name: yolo-network
mongo_port: 27017
backend_port: 5000
frontend_port: 3000
```

### Secrets (ansible/secrets.yml)

Encrypted with Ansible Vault (password: `12345`):
```bash
# View secrets
ansible-vault view ansible/secrets.yml

# Edit secrets
ansible-vault edit ansible/secrets.yml
```

---

## Usage

### Basic Commands

```bash
# Start VM and provision
vagrant up

# Re-provision existing VM
vagrant provision

# SSH into VM
vagrant ssh

# Stop VM
vagrant halt

# Destroy VM
vagrant destroy

# Check VM status
vagrant status
```

### Ansible Commands

```bash
# Run full playbook manually
ansible-playbook -i hosts playbook.yaml --ask-vault-pass

# Run specific role/stage
ansible-playbook -i hosts playbook.yaml --tags "backend" --ask-vault-pass

# Run setup stages only
ansible-playbook -i hosts playbook.yaml --tags "setup" --ask-vault-pass

# Skip specific stages
ansible-playbook -i hosts playbook.yaml --skip-tags "docker" --ask-vault-pass

# Check playbook syntax
ansible-playbook playbook.yaml --syntax-check

# Dry run (check mode)
ansible-playbook -i hosts playbook.yaml --check --ask-vault-pass
```

### Docker Commands (inside VM)

```bash
# SSH into VM
vagrant ssh

# Check running containers
docker ps

# View container logs
docker logs backend
docker logs client
docker logs mongo

# Restart containers
docker restart backend
docker restart client

# Check network
docker network inspect yolo-network

# Check volumes
docker volume ls
```

---

## Ansible Roles

### 1. install_docker
- Installs Docker CE and Docker Compose
- Configures Docker daemon
- Adds user to docker group
- **Tags**: `docker`, `setup`, `stage1`

### 2. install_git
- Installs Git for repository cloning
- **Tags**: `git`, `setup`, `stage2`

### 3. clone_repo
- Clones application from GitHub
- Sets proper permissions
- **Tags**: `repo`, `setup`, `stage3`

### 4. network_deployment
- Creates Docker bridge network
- Enables container communication
- **Tags**: `network`, `deploy`, `stage4`

### 5. backend_deployment
- Deploys MongoDB container
- Builds and deploys Node.js backend
- Creates .env configuration
- **Tags**: `backend`, `database`, `deploy`, `stage5`

### 6. frontend_deployment
- Builds and deploys React frontend
- Displays application URLs
- **Tags**: `frontend`, `deploy`, `stage6`

---

## Testing

### Test Individual Stages

```bash
# Test Docker installation only
vagrant provision -- --tags "stage1"

# Test backend deployment
vagrant provision -- --tags "stage5"

# Test frontend deployment
vagrant provision -- --tags "stage6"

# Test entire deployment chain
vagrant provision -- --tags "stage4,stage5,stage6"
```

### Verify Deployment

```bash
# Check container status
vagrant ssh -c "docker ps"

# Test backend API
curl http://localhost:5000/api/products

# Test frontend
curl http://localhost:3000

# Check MongoDB
vagrant ssh -c "docker exec mongo mongosh --eval 'show dbs'"
```

### Functional Testing

1. **Add Product Test**:
   - Navigate to http://localhost:3000
   - Click "Add Product"
   - Fill in: Name, Price, Description, Image URL
   - Submit and verify product appears

2. **Persistence Test**:
   - Add a product
   - Restart containers: `vagrant ssh -c "docker restart backend client"`
   - Refresh browser
   - Verify product still exists

3. **API Test**:
   ```bash
   # Get all products
   curl http://localhost:5000/api/products
   
   # Get specific product
   curl http://localhost:5000/api/products/<product_id>
   ```

---

## Troubleshooting

### Common Issues

#### 1. Vagrant Box Download Slow
```bash
# Download box manually first
vagrant box add geerlingguy/ubuntu2004 --box-version 1.0.4
```

#### 2. Port Already in Use
```bash
# Check what's using the port
lsof -i :3000
lsof -i :5000

# Kill the process or change port in Vagrantfile
```

#### 3. Ansible Vault Password
- Password is: `12345`
- If forgotten, you'll need to recreate secrets.yml

#### 4. Container Build Fails
```bash
# SSH into VM and check logs
vagrant ssh
docker logs backend
docker logs client

# Rebuild manually
cd /home/vagrant/yolo/backend
docker build -t yolo-backend .
```

#### 5. Frontend Can't Connect to Backend
```bash
# Check if containers are on same network
vagrant ssh -c "docker network inspect yolo-network"

# Verify backend is running
vagrant ssh -c "docker logs backend"
```

#### 6. MongoDB Connection Issues
```bash
# Check MongoDB logs
vagrant ssh -c "docker logs mongo"

# Verify MongoDB is accessible
vagrant ssh -c "docker exec mongo mongosh --eval 'db.runCommand({ ping: 1 })'"
```

### Clean Slate

If things go wrong, start fresh:

```bash
# Destroy VM
vagrant destroy -f

# Remove any lingering processes
vagrant global-status --prune

# Start fresh
vagrant up
```

---

## Documentation

### Detailed Explanations

- **EXPLANATION.md**: Docker and Docker Compose explanation from IP2
- **ANSIBLE_EXPLANATION.md**: Comprehensive Ansible deployment explanation
  - Playbook execution order
  - Role descriptions and positioning
  - Ansible modules used
  - Design decisions
  - Variables and configuration

### Key Concepts

1. **Infrastructure as Code (IaC)**: Entire infrastructure defined in code
2. **Idempotency**: Safe to run playbook multiple times
3. **Modularity**: Separate roles for each component
4. **Configuration Management**: Ansible automates all setup
5. **Containerization**: Docker ensures consistency
6. **Provisioning**: Vagrant creates identical VMs

---

## Assignment Requirements Checklist

### Stage 1: Ansible Instrumentation ✅

- ✅ Vagrant VM with Ubuntu 20.04 (geerlingguy/ubuntu2004)
- ✅ Playbook in root directory
- ✅ Variables implemented (ansible/group_vars/all.yml)
- ✅ Roles for different tasks (6 roles total)
- ✅ Blocks and tags throughout
- ✅ Clones repo from GitHub
- ✅ Sets up Docker containers
- ✅ Application runs on browser
- ✅ "Add Product" functionality works
- ✅ Data persistence verified

### Documentation ✅

- ✅ EXPLANATION.md (original Docker documentation)
- ✅ ANSIBLE_EXPLANATION.md (Ansible deployment explanation)
- ✅ README.md (comprehensive setup guide)
- ✅ Role explanations with positioning rationale
- ✅ Ansible modules documented
- ✅ Execution order explained

---

## Development Workflow

### Making Changes

1. **Modify roles**:
   ```bash
   # Edit role tasks
   vim ansible/roles/<role_name>/tasks/main.yml
   ```

2. **Update variables**:
   ```bash
   # Edit variables
   vim ansible/group_vars/all.yml
   ```

3. **Test changes**:
   ```bash
   # Re-provision with specific tags
   vagrant provision -- --tags "backend"
   ```

4. **Verify**:
   ```bash
   # Check application
   curl http://localhost:3000
   ```

### Adding New Roles

```bash
# Create role structure
mkdir -p ansible/roles/new_role/{tasks,templates,files,vars,defaults,meta}

# Create main tasks file
touch ansible/roles/new_role/tasks/main.yml

# Add role to playbook.yaml
```

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly with `vagrant provision`
5. Submit a pull request

---

## License

This project is for educational purposes as part of DevOps coursework.

---

## Author

**Sally**  
DevOps Configuration Management - IP3  
January 2026

---

## Acknowledgments

- Jeff Geerling for the Ubuntu Vagrant box
- Douglas Wangome for the reference repository
- Moringa School DevOps Program

---

## Support

For issues or questions:
1. Check [Troubleshooting](#troubleshooting) section
2. Review logs: `vagrant ssh -c "docker logs <container>"`
3. Verify with tags: `vagrant provision -- --tags "verify"`

**Happy Deploying! 🚀**
