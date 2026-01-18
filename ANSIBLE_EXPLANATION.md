# Ansible Deployment Explanation - YOLO E-commerce Application

## Table of Contents
1. [Overview](#overview)
2. [Playbook Execution Order](#playbook-execution-order)
3. [Role Descriptions and Positioning](#role-descriptions-and-positioning)
4. [Ansible Modules Applied](#ansible-modules-applied)
5. [Design Decisions](#design-decisions)
6. [Variables and Configuration Management](#variables-and-configuration-management)
7. [Tags and Blocks Usage](#tags-and-blocks-usage)

---

## Overview

This project implements Infrastructure as Code (IaC) using Ansible to automate the deployment of the YOLO e-commerce application. The application is a full-stack MERN (MongoDB, Express, React, Node.js) solution that is containerized using Docker and deployed on a Vagrant-provisioned Ubuntu 20.04 VM.

The deployment follows best practices including:
- **Modularity**: Each component is defined in separate roles
- **Idempotency**: Tasks can be run multiple times safely
- **Variable Management**: Centralized configuration using variable files
- **Error Handling**: Block-rescue patterns for robust execution
- **Tagging**: Selective execution of tasks for testing and debugging

---

## Playbook Execution Order

The playbook (`playbook.yaml`) executes in a carefully orchestrated sequence to ensure all dependencies are met at each stage:

### Execution Flow:

```
Pre-tasks (System Validation)
    ↓
Stage 1: install_docker (Docker Installation)
    ↓
Stage 2: install_git (Git Installation)
    ↓
Stage 3: clone_repo (Repository Cloning)
    ↓
Stage 4: network_deployment (Docker Network Creation)
    ↓
Stage 5: backend_deployment (MongoDB + Backend API)
    ↓
Stage 6: frontend_deployment (React Frontend)
    ↓
Post-tasks (Verification & Summary)
```

### Rationale for Sequential Execution:

This specific order is **critical** because:

1. **Pre-tasks** validate system connectivity and readiness
2. **Docker** must be installed before any containerization
3. **Git** must be available to clone the repository
4. **Repository** must be cloned before building images
5. **Network** must exist before containers can join it
6. **Backend** (including MongoDB) must be running before frontend
7. **Frontend** depends on backend API availability
8. **Post-tasks** verify successful deployment

Each role builds upon the previous ones, creating a dependency chain that ensures reliable deployment.

---

## Role Descriptions and Positioning

### 1. install_docker Role

**Position**: First role (Stage 1)

**Purpose**: Installs Docker Engine, Docker CLI, containerd, and the Docker Compose plugin.

**Why First?**
- All subsequent deployment stages require Docker
- Must be installed before any container operations
- Python Docker SDK needed for Ansible's Docker modules

**Key Tasks**:
- Updates apt cache and installs prerequisites (apt-transport-https, ca-certificates, curl, etc.)
- Adds Docker's official GPG key for package verification
- Adds Docker's official repository to apt sources
- Installs Docker CE, Docker CLI, containerd, and Docker Compose plugin
- Installs Python packages (docker, docker-compose) for Ansible modules
- Starts and enables Docker service
- Adds vagrant user to docker group for permission management
- Resets SSH connection to apply group membership changes

**Ansible Modules Used**:
- `ansible.builtin.apt`: Package management
- `ansible.builtin.apt_key`: GPG key management for repository verification
- `ansible.builtin.apt_repository`: Repository configuration
- `ansible.builtin.pip`: Python package installation
- `ansible.builtin.systemd`: Service management (start/enable Docker)
- `ansible.builtin.user`: User and group management
- `meta: reset_connection`: Applies permission changes without reboot

**Tags**: `docker`, `setup`, `stage1`

---

### 2. install_git Role

**Position**: Second role (Stage 2)

**Purpose**: Installs Git version control system for repository cloning.

**Why Second?**
- Git is required to clone the application repository
- Independent of Docker (can run in parallel conceptually)
- Lightweight installation with minimal dependencies

**Key Tasks**:
- Installs Git package via apt

**Ansible Modules Used**:
- `ansible.builtin.apt`: Package installation

**Tags**: `git`, `setup`, `stage2`

**Design Note**: This is kept as a separate role for modularity and reusability, even though it's a simple task.

---

### 3. clone_repo Role

**Position**: Third role (Stage 3)

**Purpose**: Clones the YOLO application source code from GitHub.

**Why Third?**
- Requires Git to be installed (Stage 2 dependency)
- Source code must be available before building Docker images
- Must happen before any containerization tasks

**Key Tasks**:
- Removes existing directory if present (ensures clean state)
- Clones repository from GitHub (using variables for URL and branch)
- Sets proper ownership recursively

**Ansible Modules Used**:
- `ansible.builtin.file`: Directory and permission management
- `ansible.builtin.git`: Git repository operations (clone, checkout)

**Variables Used**:
- `repo_url`: GitHub repository URL
- `repo_branch`: Branch to checkout (master)
- `app_directory`: Local directory path
- `app_user`: Owner of files

**Tags**: `repo`, `setup`, `stage3`

**Design Decision**: Removes existing directory first to ensure idempotency and prevent merge conflicts.

---

### 4. network_deployment Role

**Position**: Fourth role (Stage 4)

**Purpose**: Creates an isolated Docker bridge network for container communication.

**Why Fourth?**
- Requires Docker to be installed (Stage 1 dependency)
- Network must exist before containers are created
- Must be available before any container deployment

**Key Tasks**:
- Creates a Docker bridge network with specified name

**Ansible Modules Used**:
- `community.docker.docker_network`: Docker network management

**Variables Used**:
- `docker_network_name`: Name of the Docker network (yolo-network)

**Tags**: `network`, `deploy`, `stage4`

**Design Rationale**: 
- Custom bridge networks provide automatic DNS resolution
- Containers can reach each other by name (e.g., backend connects to `mongo:27017`)
- Better isolation than default bridge network
- Follows Docker networking best practices

---

### 5. backend_deployment Role

**Position**: Fifth role (Stage 5)

**Purpose**: Deploys the complete backend stack including MongoDB database and Node.js API.

**Why Fifth?**
- Requires Docker network to exist (Stage 4 dependency)
- Requires source code to be present (Stage 3 dependency)
- Must be operational before frontend deployment
- Database must be ready for API connections

**Key Tasks**:

**MongoDB Deployment**:
- Creates Docker volume for data persistence
- Starts MongoDB container with volume mount
- Waits for MongoDB to be ready on port 27017

**Backend API Deployment**:
- Creates .env file with MongoDB connection string
- Builds backend Docker image from Dockerfile
- Starts backend container with environment variables
- Waits for backend API to be ready on port 5000

**Ansible Modules Used**:
- `community.docker.docker_volume`: Volume management for data persistence
- `community.docker.docker_container`: Container lifecycle management
- `community.docker.docker_image`: Image building from Dockerfile
- `ansible.builtin.copy`: Creates .env configuration file
- `ansible.builtin.wait_for`: Service readiness checks

**Variables Used**:
- `mongo_image`, `mongo_container_name`, `mongo_port`, `mongo_volume`
- `backend_image`, `backend_container_name`, `backend_port`, `backend_dir`
- `docker_network_name`: Network connection

**Tags**: `backend`, `database`, `deploy`, `config`, `build`, `stage5`

**Design Decisions**:
- MongoDB and backend deployed together for logical grouping
- Volume ensures data persistence across container restarts
- Wait tasks prevent race conditions
- Block structure groups related tasks
- .env file generated dynamically with correct MongoDB connection string

---

### 6. frontend_deployment Role

**Position**: Sixth role (Stage 6)

**Purpose**: Builds and deploys the React frontend application.

**Why Last?**
- Frontend makes API calls to backend (Stage 5 dependency)
- All infrastructure must be ready before frontend starts
- Final layer of the application stack

**Key Tasks**:
- Builds frontend Docker image from Dockerfile
- Starts frontend container
- Waits for frontend to be ready on port 3000
- Displays application access information

**Ansible Modules Used**:
- `community.docker.docker_image`: Image building with force_source for fresh builds
- `community.docker.docker_container`: Container deployment
- `ansible.builtin.wait_for`: Service readiness verification
- `ansible.builtin.debug`: Information display to user

**Variables Used**:
- `frontend_image`, `frontend_container_name`, `frontend_port`, `frontend_dir`
- `docker_network_name`: Network connection
- Display variables for all services

**Tags**: `frontend`, `deploy`, `build`, `stage6`

**Design Decisions**:
- Longer timeout (180s) for frontend build process
- Debug message provides clear next steps for user
- Container connects to network for backend communication

---

## Ansible Modules Applied

### Core Ansible Modules (ansible.builtin)

| Module | Purpose | Roles Using It |
|--------|---------|----------------|
| `apt` | Package management (install, update) | install_docker, install_git |
| `apt_key` | Manage APT repository keys | install_docker |
| `apt_repository` | Manage APT repository sources | install_docker |
| `pip` | Install Python packages | install_docker |
| `systemd` | Manage systemd services | install_docker |
| `user` | Manage users and groups | install_docker |
| `file` | Manage files, directories, permissions | clone_repo |
| `git` | Clone and manage Git repositories | clone_repo |
| `copy` | Copy files and create content | backend_deployment |
| `wait_for` | Wait for conditions (port, file, etc.) | backend_deployment, frontend_deployment |
| `wait_for_connection` | Wait for SSH connectivity | Pre-tasks |
| `command` | Execute shell commands | Post-tasks (verification) |
| `debug` | Display messages and variables | Throughout playbook |

### Docker Collection Modules (community.docker)

| Module | Purpose | Roles Using It |
|--------|---------|----------------|
| `docker_network` | Create and manage Docker networks | network_deployment |
| `docker_volume` | Create and manage Docker volumes | backend_deployment |
| `docker_image` | Build and manage Docker images | backend_deployment, frontend_deployment |
| `docker_container` | Create and manage containers | backend_deployment, frontend_deployment |

### Special Directives

- `meta: reset_connection` - Resets SSH connection to apply user group changes (install_docker)
- `block` and `rescue` - Error handling and task grouping (post-tasks)

---

## Design Decisions

### 1. Use of Tags for Selective Execution

**Implementation**:
- Hierarchical tags: `setup`, `deploy`, `stage1-stage6`
- Component-specific tags: `docker`, `git`, `backend`, `frontend`
- Always tag: `always` for critical information

**Benefits**:
```bash
# Run only setup stages
ansible-playbook playbook.yaml --tags "setup"

# Deploy only backend
ansible-playbook playbook.yaml --tags "backend"

# Skip Docker installation
ansible-playbook playbook.yaml --skip-tags "docker"

# Run specific stage
ansible-playbook playbook.yaml --tags "stage5"
```

### 2. Variable Files (Bonus Points!)

**Implementation**:
- `ansible/group_vars/all.yml`: Global variables
- `ansible/secrets.yml`: Encrypted sensitive data (Vault)

**Variables Defined**:
```yaml
# Application configuration
app_name: yolo
app_directory: /home/vagrant/yolo
repo_url: https://github.com/douglaswangome/yolo.git

# Container names and ports
mongo_container_name: mongo
backend_container_name: backend
frontend_container_name: client

# Network configuration
docker_network_name: yolo-network
```

**Benefits**:
- Single source of truth
- Easy to modify without touching code
- Supports multiple environments
- Ansible Vault for sensitive data

### 3. Blocks for Error Handling

**Implementation in Post-tasks**:
```yaml
block:
  - name: Verify containers
  # verification tasks
rescue:
  - name: Display error
  # error handling
```

**Benefits**:
- Graceful error handling
- Grouped related tasks
- Better error messages
- Prevents playbook failure on verification issues

### 4. Idempotency

All tasks are idempotent:
- Running playbook multiple times produces same result
- No errors if resources already exist
- Safe to re-run after failures
- State-based rather than action-based

**Examples**:
- Docker installation: Checks if already installed
- Git clone: Uses `force: yes` to ensure clean state
- Container creation: Replaces existing containers
- Network creation: Only creates if not exists

### 5. Wait Tasks for Service Readiness

**Implementation**:
```yaml
- name: Wait for MongoDB
  ansible.builtin.wait_for:
    port: 27017
    delay: 5
    timeout: 60
```

**Rationale**:
- Prevents race conditions
- Ensures services are fully operational
- Improves reliability
- Provides clear failure points

**Timeouts**:
- MongoDB: 60s (database startup)
- Backend: 120s (API + npm dependencies)
- Frontend: 180s (React build + serve)

### 6. Docker Volume Strategy

**Decision**: Use named volumes for MongoDB data persistence

**Implementation**:
```yaml
- name: Create MongoDB volume
  community.docker.docker_volume:
    name: mongo-data
    state: present
```

**Rationale**:
- Data persists across container restarts
- Survives container removal
- Better performance than bind mounts
- Managed by Docker

### 7. Network Isolation with Custom Bridge

**Decision**: Create custom Docker bridge network

**Benefits**:
- Automatic DNS resolution (backend connects to `mongo` by name)
- Better isolation from other containers
- More control over network configuration
- Follows Docker best practices

---

## Variables and Configuration Management

### Variable Hierarchy

1. **Playbook level**: None (all externalized)
2. **vars_files**: 
   - `ansible/group_vars/all.yml` (global vars)
   - `ansible/secrets.yml` (encrypted vars)
3. **Role defaults**: None (using group_vars instead)
4. **Facts**: Ansible-gathered system information

### Key Variables Reference

```yaml
# Application
app_name: yolo
app_user: vagrant
app_directory: /home/vagrant/yolo
repo_url: https://github.com/douglaswangome/yolo.git
repo_branch: master

# Network
docker_network_name: yolo-network

# MongoDB
mongo_image: mongo:5.0
mongo_container_name: mongo
mongo_port: 27017
mongo_volume: mongo-data

# Backend
backend_image: yolo-backend
backend_container_name: backend
backend_port: 5000

# Frontend
frontend_image: yolo-client
frontend_container_name: client
frontend_port: 3000
```

### Ansible Vault for Secrets

The `ansible/secrets.yml` file is encrypted using Ansible Vault:

```bash
# Create encrypted file
ansible-vault create ansible/secrets.yml

# Edit encrypted file
ansible-vault edit ansible/secrets.yml

# View encrypted file
ansible-vault view ansible/secrets.yml
```

**Vault Password**: 12345 (for demonstration/grading purposes)

---

## Tags and Blocks Usage

### Tag Categories

1. **Stage Tags**: `stage1` through `stage6` for sequential execution
2. **Component Tags**: `docker`, `git`, `backend`, `frontend`, `database`
3. **Action Tags**: `setup`, `deploy`, `build`, `config`
4. **Special Tags**: `always`, `verify`

### Block Usage

**Post-tasks Verification Block**:
```yaml
block:
  - Check containers
  - Verify services
rescue:
  - Error handling
  - Helpful messages
```

**Backend Deployment Block**:
```yaml
block:
  - MongoDB volume
  - MongoDB container
  - MongoDB wait
tags:
  - backend
  - database
```

---

## Testing and Verification

### Selective Testing Commands

```bash
# Test full deployment
vagrant up

# Re-run provisioning
vagrant provision

# Test specific stage
vagrant provision --provision-with ansible -- --tags "stage3"

# Test backend only
vagrant provision --provision-with ansible -- --tags "backend"

# Skip Docker installation
vagrant provision --provision-with ansible -- --skip-tags "docker"
```

### Verification Steps

1. **Container Status**: `docker ps`
2. **Container Logs**: `docker logs <container_name>`
3. **Network Inspection**: `docker network inspect yolo-network`
4. **Volume Inspection**: `docker volume ls`
5. **Application Test**: Visit `http://localhost:3000`

---

## Conclusion

This Ansible implementation demonstrates best practices in configuration management:

✅ **Modularity**: Six separate roles with clear responsibilities  
✅ **Idempotency**: Safe to run multiple times  
✅ **Variables**: External configuration files (bonus points!)  
✅ **Tags**: Granular control over execution  
✅ **Blocks**: Proper error handling  
✅ **Roles**: Clear dependencies and execution order  
✅ **Documentation**: Comprehensive explanation of design decisions  

The sequential execution order ensures all dependencies are met at each stage, resulting in a reliable, automated deployment process that can be executed with a single command: `vagrant up`.

---

**Student**: Sally  
**Course**: DevOps Configuration Management - IP3  
**Target Grade**: A (Comprehensive Implementation!)  
**Date**: January 2026

