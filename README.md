# learning-ansible
random notes and files created while learning ansible

### commands
ansible --list-hosts <GROUP>                          : shows all the hosts that the machine knows about, values
                                                  : can be all, groupname, wildcards, hostnames
ansible -i inventory.file --list-hosts all        : use the -i option to state the inventory file


- you have 2 ways in which you can define a inventory file (inventory holds information about nodes/machines under management)
  - static; as in a defined file with all the machines
  - dynamic; a programme, typically a python script will run before that will generate the file
- the default file that contains all the hosts lives in /etc/ansible/hosts
  - use the command: ansible --list-hosts all to view all the hosts
- its best not to use the /etc/ansible/hosts file but to define your own that can be checked into source control
- the file has 2 formats
  - basic list of all machines ip's or names
  - grouping of each sub list
- file should also contain the control/master server
```
[dev]
xxx.xxx.xxx.xxx
yyy.yyy.yyy.yyy

[stage]
zzz.zzz.zzz.zzz
db.server

[local]
master ansible-connection=local #do not ssh into this machine as your already there
```
- in the inventory file, you can specify a number of things, one of which is a connection type which defines how to ssh into each of those machines
- /etc/ansible/ansible.cfg contains all the config, one of which is the inventory option that contains the file to first look at for the inventory
  - you can either make changes there or you can override them using a local config file with the overridden options
  - e.g. define a ansible.cfg in your home directory with the customisation and point to that

```ansible.cfg in local directory
[defaults]
inventory = ./hosts
```
