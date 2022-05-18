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