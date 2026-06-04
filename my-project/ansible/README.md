# Ansible deployment

In this part I used Ansible to deploy the Docker application on the remote server.

The server used in the inventory is:

```
jatin.khiyani.takima.school
```

## Folder content

```
my-project/ansible/
├── inventories/setup.yml
├── group_vars/all.yml
├── playbook.yml
└── roles/
    ├── docker/
    ├── network/
    ├── database/
    ├── app/
    ├── frontend/
    └── proxy/
```

## 3-1 Document your inventory and base commands

Inventory file:

```
my-project/ansible/inventories/setup.yml
```

Content:

```
all:
  vars:
    ansible_user: admin
    ansible_python_interpreter: /opt/docker_venv/bin/python
  children:
    prod:
      hosts:
        jatin.khiyani.takima.school:
```

The user is `admin` because the remote server uses this Debian user.

Base commands:

```
cd my-project/ansible
ansible all -i inventories/setup.yml -m ping
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

The ping command checks that Ansible can connect to the server.

The setup command reads facts from the server.

The apt command removes Apache that was installed in the discovery part.

## group_vars

The shared variables are in:

```
my-project/ansible/group_vars/all.yml
```

It contains DockerHub username, database credentials and internal container names.

```
DOCKERHUB_USERNAME: jatinkhiyani
POSTGRES_DB: db
POSTGRES_USER: usr
POSTGRES_PASSWORD: pwd
DB_HOST: database
BACKEND_HOST: backend
FRONTEND_HOST: frontend
```

For real production, passwords should be moved to a vault or GitHub Secrets.

## 3-2 Document your playbook

The main playbook is:

```
my-project/ansible/playbook.yml
```

Content:

```
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
    - network
    - database
    - app
    - frontend
    - proxy
```

Run the playbook:

```
ansible-playbook -i inventories/setup.yml playbook.yml
```

Check the playbook syntax:

```
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
```

The playbook installs Docker and then deploys each part of the application as a container.

## Roles

### docker

The `docker` role installs required packages, adds Docker GPG key, adds Docker repository, installs Docker packages, creates a Python virtual environment and installs the Docker SDK.

It also starts and enables Docker.

### network

The `network` role creates the Docker network:

```
my-network
```

This network is used by database, backend, frontend and proxy.

### database

The `database` role launches the Postgres container from DockerHub.

### app

The `app` role launches the Springboot backend.

### frontend

The `frontend` role launches the Vue/Nginx front-end.

### proxy

The `proxy` role launches the Apache httpd reverse proxy and exposes port 80.

## 3-3 Document your docker_container tasks configuration.

Database container:

```
community.docker.docker_container:
  name: database
  image: "{{ DOCKERHUB_USERNAME }}/tp-devops-database:latest"
  state: started
  restart_policy: unless-stopped
  pull: true
  networks:
    - name: my-network
  env:
    POSTGRES_DB: "{{ POSTGRES_DB }}"
    POSTGRES_USER: "{{ POSTGRES_USER }}"
    POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
```

Backend container:

```
community.docker.docker_container:
  name: backend
  image: "{{ DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest"
  state: started
  restart_policy: unless-stopped
  pull: true
  networks:
    - name: my-network
  env:
    DB_HOST: "{{ DB_HOST }}"
    DB_NAME: "{{ POSTGRES_DB }}"
    DB_USER: "{{ POSTGRES_USER }}"
    DB_PASSWORD: "{{ POSTGRES_PASSWORD }}"
```

Frontend container:

```
community.docker.docker_container:
  name: frontend
  image: "{{ DOCKERHUB_USERNAME }}/frontend:latest"
  state: started
  restart_policy: unless-stopped
  pull: true
  recreate: true
  networks:
    - name: my-network
```

Proxy container:

```
community.docker.docker_container:
  name: httpd
  image: "{{ DOCKERHUB_USERNAME }}/tp-devops-httpd:latest"
  state: started
  restart_policy: unless-stopped
  pull: true
  networks:
    - name: my-network
  ports:
    - "80:80"
  env:
    BACKEND_HOST: "{{ BACKEND_HOST }}"
    FRONTEND_HOST: "{{ FRONTEND_HOST }}"
```

## Continuous Deployment

The GitHub Actions deployment file is:

```
.github/workflows/deploy.yml
```

It installs Ansible, installs the Docker collection, creates a private key file from GitHub Secrets and runs the playbook.

## Is it really safe to deploy automatically every new image on the hub? explain. What can I do to make it more secure?

It is not fully safe to deploy automatically every new image from DockerHub.

A bad image could be pushed by mistake, or an image could be compromised. If the server deploys it directly, production can break.

To make it more secure I can:

1. Deploy only after tests and Sonar are green.
2. Use protected branches.
3. Use GitHub environments with manual approval.
4. Use version tags instead of only `latest`.
5. Scan images before deployment.
6. Keep SSH keys and passwords in GitHub Secrets or Ansible Vault.

