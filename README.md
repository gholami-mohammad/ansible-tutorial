# Preparing
1. ssh to all servers to accept ssh fingerprint `ssh user@server_ip`
1. generate ssh-key `ssh-keygen -t ed25519 -C "mgh key for ansible" -f ~/.ssh/ansible` 
1. copy ssh public key to servers: `ssh-copy-id -i ~/.ssh/ansible.pub user@server_ip`

# Install Ansible
```
sudo apt update
sudo apt install ansible
```
# Config ansible

## Create inventory file
create `inventory` file and add servers' IP addresses to the file.

```
touch inventory
tee -a inventory <<EOF
server_1_ip
server_2_ip
server_3_ip
example.com
EOF
```

## Ping inventory
```
ansible all --private-key ~/.ssh/ansible -i inventory -m ping
```
You have to get a success message for all your servers.

## Create config file
re-check the ping command with a local config file:
```
ansible all -m ping
```
## Get all information about servers
```
ansible all -m gather_facts | less

# limited to a host
ansible all -m gather_facts --limit server_ip | less

```

## Run command that needs sudo permission
```
ansible all -m apt -a update_cache=true --become --ask-become-pass
```
- **--become** causes to run the command as `sudo`
- **--ask-become-pass** take sudo password

## Install a simple app using apt
```
ansible all -m apt -a name=nginx --become --ask-become-pass
```

or this command to do dist-upgrade:
```
ansible all -m apt -a "upgrade=dist" --become --ask-become-pass
```

# Creating playbook

### creating a playbook to install Nginx:
```
touch nginx.yml

tee -a nginx.yml <<EOF
---
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes

  - name: install nginx package
    apt:
      name: nginx
      state: latest # install the latest version
EOF
```

to install the created playbook:
```
ansible-playbook --ask-become-pass nginx.yml
```

### Creating a playbook to remove apache2 (if installed)

```
touch remove_apache2.yml
tee -a remove_apache2.yml <<EOF
---
- hosts: all
  become: true
  tasks:
    - name: remove apache2
      apt:
        name: apache2
        state: absent # <- the absent state removes the package 
EOF

ansible-playbook --ask-become-pass remove_apache2.yml
```

---
## Using `when` in playbook
For each task, you can add the `when` command to run if the condition is true.
For example, run a task if the server is Ubuntu: `when: ansible_distribution == "Ubuntu"` or check if server is Ubuntu or Debian: `when: ansible_distribution in ["Ubuntu" , "Debian"]`.
Every key that returns from `ansible all -m gather_facts` can be used in `when` command.

Install Nginx on ubuntu and centos server:
```
---
- hosts: all
  become: true
  tasks:

  - name: update repository index
    when: ansible_distribution == "Ubuntu"
    apt:
      update_cache: yes

  - name: install nginx package
    when: ansible_distribution == "Ubuntu"
    apt:
      name: nginx
      state: latest # install the latest version

  - name: update repository index
    when: ansible_distribution == "CentOS"
    dnf:
      update_cache: yes

  - name: install nginx package
    when: ansible_distribution == "CentOS"
    dnf:
      name: nginx
      state: latest # install the latest version
```

---
## Refactoring playbook
Refactoring the last playbook:
```
---
- hosts: all
  become: true
  tasks:

  - name: install nginx package
    when: ansible_distribution == "Ubuntu"
    apt:
      name: nginx
      state: latest # install the latest version
      update_cache: yes # <- update_cache is an option for apt so it will update cache before installing nginx

  - name: install nginx package
    when: ansible_distribution == "CentOS"
    dnf:
      name: nginx
      state: latest # install the latest version
      update_cache: yes
```

---
## Using variables in the playbook file

Example: Install Apache package on Ubuntu and CentOS. Ubuntu package name is apache2 and CentOS package name is `httpd`.
```
---
- hosts: all
  become: true
  tasks:
  - name: install apache
    package: # <- package is a generic package manager that chooses the correct package manager based on OS.
      name: "{{ apache_package }}"
      state: latest
      update_cache: yes
```

And edit your inventory to add variables:
```
192.168.1.2 apache_package=apache2 # <- ubuntu server
192.168.1.3 apache_package=httpd # <- centos server
```

## Grouping nodes
In the inventory file, you can group nodes like the example below. One node can be in multiple groups as well.

inventory file:
```
[web_servers]
192.168.1.67
192.168.1.68
192.168.1.11

[db_servers]
192.168.1.98
192.168.1.99
192.168.1.11

[cache_servers]
192.168.1.33
192.168.1.11

[testing_servers]
192.168.1.11
```

## Targeting specific nodes
When the inventory file contains multiple groups, in the YAML file you can target specific nodes like this: (full example exists in the site.yml file)
```
---
- hosts: web_servers
``` 

## Tags
By adding tags to plays and tasks, you can run tasks or plays with specific tags. There are 2 special tags: `always` and `never` for more information read [this](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#special-tags-always-and-never) article.

```
---

- hosts: all
  tags:
    - always
  become: true
  tasks:
    - name: Install updates(CentOS servers)
      when: ansible_distribution == "CentOS"
      tags:
        - always # <- always run this task
      dnf:
        update_only: yes
        update_cache: yes
    
    - name: Install updates(Ubuntu servers)
      when: ansible_distribution == "Ubuntu"
      tags:
        - always # <- always run this task
      apt:
        update_cache: yes
        upgrade: dist
      

- hosts: web_servers
  become: true
  tasks:
  - name: install nginx package
    tags:
      - nginx
      - centos
      - ubuntu
    package: # <- selects the right package manager base on the target OS
      name: nginx
      state: latest
  - name: install php package
    tags:
      - php
      - ubuntu
    package:
      name: php
      state: latest
  - name: install git package
    tags:
      - git
      - ubuntu
      - centos
    package:
      name: git
      state: latest
```

### list tags of a playbook
```
ansible-playbook --list-tags site.yml
```

### Run playbook against specific tags
file option to use tags to run playbooks: [reference](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#selecting-or-skipping-tags-when-you-run-a-playbook)

* `--tags all` - run all tasks, ignore tags (default behavior)

* `--tags [tag1, tag2]` - run only tasks with either the tag tag1 or the tag tag2

* `--skip-tags [tag3, tag4]` - run all tasks except those with either the tag tag3 or the tag tag4

* `--tags tagged` - run only tasks with at least one tag

* `--tags untagged` - run only tasks with no tags

example:
```
ansible-playbook --tags "nginx,ubuntu" --ask-become-pass site.yml
```
This will run playbooks for tasks that have `nginx` or `ubuntu` tag.

## Copying files
To copy files to the server, create a directory named `files` and put your files there.
```
---
- hosts: all
  tasks:
    - name: copy default html file for web servers
      copy:
        src: default_site.html
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: 0644
```

## Managing services
To start, stop, restart, enable, or disable a service, you can add a play like this:
```
---
- hosts: all
  become: true
  tasks:
    - name: start nginx service
      service:
        name: nginx
        state: started # it is equal to 'sudo systemctl start nginx' command in centos and ubuntu
        enabled: yes # it is equal to 'sudo systemctl enable nginx' command in centos and ubuntu
```
The value of `state` could be one of these 4 options:
- reloaded
- restarted
- started
- stopped

## Registering a change on a task
You want to register a state change after a task was executed, and then use that state in other tasks:
```
---
- hosts: all
  become: true
  tasks:
    - name: copy nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: 0644
      register: nginx_updated # <- here we registered a state named nginx_updated and it will get true if this task marked as changed
    
    - name: restart nginx
      when: nginx_updated.changed # <- using registered state here
      service:
        name: nginx
        state: restarted
```

## Adding User
A good example of creating a limited user and sudoer user added to the [bootstrap.yml](./bootstrap.yml) file.

## Roles
The roles are for breaking the playbook into multiple files. So they are easier to read and maintain and also make the files reusable.

To create tasks for roles, you must create this directory structure: `roles/ROLE_NAME/tasks/main.yml`.
To use these roles in the playbook, simply use them like this:
```
---
- hosts: all
  become: true
  roles:
    - a_role_name # <- it points to a file in `roles/a_role_name/tasks/main.yml`
```

The example of using roles is added in the [site_using_roles.yml](./site_using_roles.yml) file. (This example was implemented for Ubuntu servers and  tested on Ubuntu 20.04)

## Handlers

Handlers are special tasks that run at the end of a playbook run. They must call using the `notify` keyword.

Handlers could be placed in a directory name handlers in the root directory or root directory of each role.

The example of the handles was added to the previous example to restart the Nginx service in the previous example.

## Host variables
If you are using variables in the playbook and they are different in each host, the best way of managing variables is the host variable.

First, create a directory named host_vars in the root directory. To define the variables of each node, you should create a file with the name of the node with the .yml extension. For example, if the server node in the inventory file is 192.168.1.147, so the host vars file must be `192.168.1.147.yml`.

Example:
```
---
- hosts: all
  become: true
  tasks:
    - name: add default user
      user:
        name: "{{ username }}"
        create_home: "{{ create_home }}"
        state: present
        uid: "{{ user_uid }}"
```
The `host_vars/192.168.1.147.yml` looks like this:
```
username: example
create_home: yes
user_uid: 123
```
And `host_vars/192.168.1.150.yml` looks like this:
```
username: example
create_home: no
user_uid: 999
```

## Templates
Templates are used for creating predefined files on nodes based on the variables of each host. It uses Jinja2 templating system.

Template files are placed in a directory called `templates` in each role's root directory with `.j2` extension.

Let's create the pg_hba.conf file for the Postgresql using template.

```
# roles/postgresql/templates/pg_hba.conf.j2
local   all                         postgres                                peer

# TYPE  DATABASE                    USER            ADDRESS                 METHOD
local   all                         all                                     peer

# IPv4 local connections:
host    {{ db_name }}             {{ username }}    127.0.0.1/32            {{ encryption }}

# IPv6 local connections:
host    {{ db_name }}             {{ username }}    ::1/128                 {{ encryption }}

local   replication                 all                                     peer
host    replication                 all             127.0.0.1/32            {{ encryption }}
host    replication                 all             ::1/128                 {{ encryption }}
```

And the playbook is [postgresql](./roles/postgresql/tasks/main.yml) that this template added to it.