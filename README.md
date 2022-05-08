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
create `inventory` file and add servers IP addresses to the file.

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
You have to get success message for all your servers.

## Create config file
```
touch ansible.cfg

tee -a ansible.cfg <<EOF
[defaults]
# inventory file path
inventory = inventory
private_key_file = ~/.ssh/ansible
EOF
```

re-check ping command with local config file:
```
ansible all -m ping
```
## Get all information about servers
```
ansible all -m gather_facts | less

# limited to a host
ansible all -m gather_facts --limit server_ip | less

```