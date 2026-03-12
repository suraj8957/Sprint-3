# SonarQube Authorization Setup using Ansible Role
---
## 1. Overview
This document describes the complete setup for configuring **Role-Based Authorization (RBAC)** in SonarQube and automating the configuration using **Ansible roles**.
The goal of this setup is to:
- Create and manage SonarQube groups
- Assign permissions to Dev, QA, and DevOps teams
- Explore SonarQube extensions
- Show how SonarQube configuration can be managed using Ansible automation

---

## 2. Architecture
<img width="576" height="374" alt="_- visual selection (20)" src="https://github.com/user-attachments/assets/ea47c54b-6c81-4204-ab1c-1b8b06acc8f9" />

---

# 3. Prerequisites

Before starting, ensure the following:

- Ubuntu/Linux server
- SonarQube installed and running
- Ansible installed on control node
- Admin access to SonarQube
- Git repository available for testing

---

## 4. Create Ansible Role

#### Step-1 Create a new Ansible role for SonarQube.
```bash
ansible-galaxy init <role_name>
```
**Role Structure:**
<img width="1850" height="578" alt="image" src="https://github.com/user-attachments/assets/c64c7416-e06e-4589-a210-bc5282f2061e" />

**inventory.ini**
```
[sonarqube]
sonar ansible_host=15.206.28.132 ansible_user=ubuntu ansible_ssh_private_key_file=/home/suraj/Downloads/ot_micro_service.pem


