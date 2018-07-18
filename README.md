# learning-ansible
random notes and files created while learning ansible

### commands
ansible --list-hosts <GROUP>                          : shows all the hosts that the machine knows about, values
                                                  : can be all, groupname, wildcards, hostnames
ansible -i inventory.file --list-hosts all        : use the -i option to state the inventory file
ansible -m ping all                               : ping all hosts
ansible -m command -a "hostname" all              : get the hostname of all inventory machines. -a "hostname" is the argument
ansible-playbook <playbook_file.yml>              : run a playbook

### Inventory
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

### Tasks
- A task is the most basic building block of all execution and configuration
- A task is made out of 2 parts
  - a module
  - arguments for the module
- In the ping example '-m' states module, 'ping' is the module
- The command module runs a command thats provided as the argument
  - The command module is the default command if no module is provided
  - ansible -a "hostname" all : will run the command module
- output of commands are:
  - host | status | exit code

```
  paul@master:~/learning-ansible$ ansible -m command -a "hostname" all
  127.0.0.1 | SUCCESS | rc=0 >>
  master
```
- To view the modules available look at the documentation online

### Plays
- A playbook is a file in yaml made up of plays
- A play is a set of target hosts and a task to execute against those hosts
- playbooks start out with a list of plays
  - 2 main keys for a play
    - hosts
    - tasks : which is also a list
- To run a playbook, you use the command: ansible-playbook <playbook_file.yml>
```
---
  - hosts: all # the first play
    tasks:
    - name: get server hostname # a nice description of the task
      command: hostname # the module to run
```
- When running a playbook, you sometimes need sudo/root level privs, to escalate privs' you need to add the become property
at the host level set to true

```
---
  - hosts: all # the first play
    become: true
    tasks:
```

















 =
