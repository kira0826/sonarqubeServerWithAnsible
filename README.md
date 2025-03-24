## Kevin Steven Nieto Curaca - A00395466

## SonarQube Setup and Automation Process

First of all, according to the SonarQube documentation, we need to establish a database with a specific DB, schema, and user that will be used by the SonarQube server. In this case, the server will be deployed using the .zip strategy. To achieve this, I used PostgreSQL as the database and automated its installation, configuration, and startup using the following role:

```
- name: Install PostgreSQL
  apt:
    name: postgresql  # Install PostgreSQL package
    state: present    # Ensure it is installed
    update_cache: yes # Update the package cache before installing

- name: Ensure PostgreSQL service is running
  service:
    name: postgresql  # Name of the service
    state: started     # Ensure the service is running
    enabled: yes       # Enable the service to start on boot

- name: Install python3-psycopg2
  apt:
    name: python3-psycopg2  # Install psycopg2 (required for PostgreSQL modules in Ansible)
    state: present          # Ensure it is installed
    update_cache: yes       # Update the package cache before installing

- name: Create sonarqube user on PostgreSQL
  become: yes  # Run as root
  command: >
    sudo -u postgres sh -c "cd /tmp && psql -c \"CREATE USER sonarqube WITH PASSWORD 'sonarqube' CREATEDB LOGIN;\""
  args:
    creates: /var/lib/postgresql/.pgpass  # Prevent task from running if the user already exists
  register: create_user_result            # Store the result of the command
  ignore_errors: yes                      # Ignore errors if the user already exists
  changed_when: "'already exists' not in create_user_result.stderr"  # Mark as changed only if the user was created

- name: Create sonarqube database on PostgreSQL
  become: yes  # Run as root
  command: >
    sudo -u postgres sh -c "cd /tmp && psql -c \"CREATE DATABASE sonarqube OWNER sonarqube;\""
  args:
    creates: /var/lib/postgresql/.pgpass  # Prevent task from running if the database already exists
  register: create_db_result              # Store the result of the command
  ignore_errors: yes                      # Ignore errors if the database already exists
  changed_when: "'already exists' not in create_db_result.stderr"  # Mark as changed only if the database was created

- name: Create empty schema for SonarQube
  become: yes  # Run as root
  command: >
    sudo -u postgres sh -c "cd /tmp && psql -d sonarqube -c \"CREATE SCHEMA sonarqube AUTHORIZATION sonarqube;\""
  args:
    creates: /var/lib/postgresql/.pgpass  # Prevent task from running if the schema already exists
  register: create_schema_result          # Store the result of the command
  ignore_errors: yes                      # Ignore errors if the schema already exists
  changed_when: "'already exists' not in create_schema_result.stderr"  # Mark as changed only if the schema was created

- name: Grant permissions to sonarqube user on the schema
  become: yes  # Run as root
  command: >
    sudo -u postgres sh -c "cd /tmp && psql -d sonarqube -c \"
      GRANT ALL PRIVILEGES ON SCHEMA sonarqube TO sonarqube;
      GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA sonarqube TO sonarqube;
      GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA sonarqube TO sonarqube;\""
  args:
    creates: /var/lib/postgresql/.pgpass  # Prevent task from running if permissions are already granted
  register: grant_permissions_result      # Store the result of the command
  ignore_errors: yes                      # Ignore errors if permissions already exist
  changed_when: "'already exists' not in grant_permissions_result.stderr"  # Mark as changed only if permissions were granted                                 

```

While setting up PostgreSQL, I encountered an issue: executing PostgreSQL commands as the default user did not work due to the database authentication process. To bypass this, all commands were executed using sudo -u postgres, which allowed them to run with root privileges.

Additionally, since I was working with an Ubuntu 22.04 LTS (Gen 2) virtual machine, I found that Python3-psycopg2 was not installed by default. This package is required to use PostgreSQL modules with Ansible, so I had to install it manually.

## Java Installation & SonarQube Setup

Since SonarQube requires Java 17, I created a dedicated Ansible role to install it. After that, I proceeded with the SonarQube setup, which involved using the official SonarQube repositories to install the required package. For this setup, I opted for SonarQube version 9.9.1.69595.

![image](https://github.com/user-attachments/assets/f27c96cb-a2f9-4075-a2ce-0e195d304ae9)


To make the configuration flexible, I stored all relevant version numbers and other volatile information in a defaults file as variables, making it easier to update in the future:

![image](https://github.com/user-attachments/assets/b0bea934-4437-4547-91cf-a4896acce296)

Additionally, I had to use the creates argument in Ansible tasks to prevent issues when a resource was already created.

```
- name: Rename SonarQube directory
  command: >
    mv {{ sonarqube_home }}/sonarqube-{{ sonarqube_version }} {{ sonarqube_home }}/sonarqube
  args:
    creates: "{{ sonarqube_home }}/sonarqube"  # Evita renombrar si ya existe

```

A key step in the setup was configuring sonar.properties, where I linked the SonarQube server to the previously set up PostgreSQL database : 

```
- name: Configure sonar.properties
  lineinfile:
    path: "{{ sonarqube_home }}/sonarqube/conf/sonar.properties"
    regexp: "^{{ item.key }}="  # Busca la línea que comienza con la clave
    line: "{{ item.key }}={{ item.value }}"  # Define la línea completa
    create: yes  # Crea el archivo si no existe
    state: present
  loop:
    - { key: "sonar.jdbc.username", value: "{{ postgres_username }}" }
    - { key: "sonar.jdbc.password", value: "{{ postgres_password }}" }
    - { key: "sonar.jdbc.url", value: "jdbc:postgresql://localhost/{{ postgres_db }}" }
```

## Virtual Memory Issue & Fix

During deployment, I encountered a significant issue related to virtual memory allocation on the virtual machine. SonarQube requires a minimum memory allocation that exceeded the default limit set by the VM. To resolve this, I had to adjust the memory allocation using the following command:

```
# Set vm max for elastic search
- name: Set vm.max_map_count for current session
  become: yes
  command: sysctl -w vm.max_map_count=262144

- name: Persist vm.max_map_count setting
  become: yes
  lineinfile:
    path: /etc/sysctl.conf
    regexp: "^vm.max_map_count"
    line: "vm.max_map_count=262144"
    create: yes

- name: Reload sysctl settings
  become: yes
  command: sysctl -p
```
## Final Steps: Server Startup & Repository Cloning

Once the configuration was complete, I simply needed to start the SonarQube server inside the role using: 

```
- name: Start SonarQube
  become: yes
  become_user: azureuser # Must select the owner user to perform the command  
  command: >
    {{ sonarqube_home }}/sonarqube/bin/linux-x86-64/sonar.sh start
  args:
    chdir: "{{ sonarqube_home }}/sonarqube/bin/linux-x86-64"
```

To further automate the process, I created an Ansible role to clone the required repository automatically. This ensured that the latest version of the code was always available.

```
- name: Ensure SonarQube directory exists
  become: yes
  become_user: azureuser
  file:
    owner: "azureuser"
    path: "/home/azureuser/repository"
    state: directory
    mode: '0750'

- name: Install Git
  apt:
    name: git
    state: present
  when: ansible_os_family == "Debian"

- name: Clone Git Repository
  become: yes
  become_user: azureuser
  git:
    repo: "https://github.com/CMR-SID2/backend"
    dest: "/home/azureuser/repository"
    version: "main"
    force: yes
```
Finally, I made a big role that contains all roles in the correct sequence to be execute:

```
---
- name: Run all roles
  hosts: azure_vm 
  become: true
  roles:
    - setup_java
    - install_sonarqube_database
    - install_and_run_sonarqube
    - git_install
```
### Screenshots

![Screenshot From 2025-03-23 17-22-28](https://github.com/user-attachments/assets/6a384443-30f8-42eb-9feb-16d8a87ba8b3)

![Screenshot From 2025-03-23 17-22-39](https://github.com/user-attachments/assets/f4220138-2bf0-49bc-bfff-b263eb90cb77)

![Screenshot From 2025-03-23 17-23-03](https://github.com/user-attachments/assets/05716a68-9f47-4a8a-aa16-771bbab2e288)

Note: To avoid problems with gradle 9.0 we need to increase sonar version plugin based on the version number display on: 

https://docs.sonarsource.com/sonarqube-server/latest/analyzing-source-code/scanners/sonarscanner-for-gradle/

