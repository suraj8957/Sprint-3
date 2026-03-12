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

#### Step 1: Create a new Ansible role for SonarQube.
```bash
ansible-galaxy init <role_name>
```
**Role Structure:**
<img width="1850" height="578" alt="image" src="https://github.com/user-attachments/assets/c64c7416-e06e-4589-a210-bc5282f2061e" />

**inventory.ini**
```
[sonarqube]
sonar ansible_host=15.206.28.132 ansible_user=ubuntu ansible_ssh_private_key_file=/home/suraj/Downloads/ot_micro_service.pem
```
**Playbook install-sonarqube.yml**
```
---
- hosts: sonarqube
  become: yes

  roles:
    - sonarcube
```
**default/main.yml**
```
sonarqube_version: "10.4.0"
sonarqube_user: "sonar"
sonarqube_group: "sonar"
sonarqube_port: 9000
sonarqube_db_name: "sonarqube"
sonarqube_db_user: "sonar"
sonarqube_db_password: "sonar"
```
**tasks/main.yml**
```
---
# install Java 
- name: Install Java
  apt:
    name: openjdk-17-jdk
    state: present
    update_cache: yes

# install unzip 

- name: Install unzip
  apt:
    name: unzip
    state: present

# Install PostgreSQL
- name: Install PostgreSQL
  apt:
    name: postgresql
    state: present

# Install psycopg2
- name: Install PostgreSQL Python library
  apt:
    name: python3-psycopg2
    state: present

# create database
- name: Create sonar database
  become_user: postgres
  postgresql_db:
    name: sonarqube

- name: Create sonar user
  become_user: postgres
  postgresql_user:
    name: sonar
    password: sonar
    state: present

# Grant privileges
- name: Grant privileges
  become_user: postgres
  postgresql_privs:
    database: sonarqube
    roles: sonar
    privs: ALL
    type: database

# Create SonarQube user
- name: Create sonar user
  user:
    name: sonar
    shell: /bin/bash

# Download SonarQube
- name: Download SonarQube
  get_url:
    url: https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
    dest: /opt/sonarqube.zip

# Unzip SonarQube
- name: Unzip SonarQube
  unarchive:
    src: /opt/sonarqube.zip
    dest: /opt/
    remote_src: yes

- name: Rename SonarQube folder
  command: mv /opt/sonarqube-10.4.1.88267 /opt/sonarqube
  args:
    creates: /opt/sonarqube

# Change ownership
- name: Change ownership
  file:
    path: /opt/sonarqube
    owner: sonar
    group: sonar
    recurse: yes

- name: Configure SonarQube
  template:
    src: sonar.properties.j2
    dest: /opt/sonarqube/conf/sonar.properties
    
- name: Set vm.max_map_count
  sysctl:
    name: vm.max_map_count
    value: "262144"
    state: present

- name: Start SonarQube
  shell: sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

**templates/sonar.properties.j2**
```
sonar.jdbc.username={{ sonarqube_db_user }}
sonar.jdbc.password={{ sonarqube_db_password }}
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

#### Step 2: Access SonarQube

Open SonarQube in the browser:
```
http://<server-ip>:9000
```
Login:
```
Username: admin
Password: admin
```
**Note:** By default, Username and Password are admin
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a4a0a777-5c4b-4e54-a8e6-fdd180c0ff6b" />

#### Step 3: Create Groups in SonarQube
```
Administration
→ Security
→ Groups
```
<img width="1920" height="430" alt="image (4)" src="https://github.com/user-attachments/assets/a9d0adb8-29e3-415c-8cef-f01a75598c86" />

**Create the following groups:**

```
Dev 
QA
DevOps
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/966dbc67-8636-4831-9482-5cb4e83ef29f" />

#### Step 4: Create Users

**Navigate to:**

```
Administration
→ Security
→ Users
```

Create users and assign them to groups.

| User | Group |
|-----|------|
| devuser | Dev |
| qauser | QA |
| devopsuser | DevOps |
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/47720097-d035-4e5f-8e16-89da22f263cf" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/300a2296-2b88-4205-a5fe-a45ccbb8f10c" />

#### # Step 5: Assign Permissions
Open the project:

```
Projects
→ employee-api
→ Project Settings
→ Permissions
```
Assign permissions:
| Group | Permission |
|------|-------------|
| Dev | Execute Analysis |
| QA | Execute Analysis |
| DevOps | Administer |
<img width="1920" height="820" alt="image" src="https://github.com/user-attachments/assets/929684d8-ae8a-441b-a1aa-3850a5f209a3" />


Verify permissions for users

**Dev User:**
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ca3570e7-3366-42ce-b3b9-1f4614c9de41" />

**QA User:**
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d1a53fbd-ad35-4468-9078-e3635304d796" />

**DevOps User:**












