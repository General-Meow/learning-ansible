# learning-ansible
random notes and files created while learning ansible

### commands
ansible --list-hosts <GROUP>                          : shows all the hosts that the machine knows about, values
                                                  : can be all, groupname, wildcards, hostnames
ansible -i inventory.file --list-hosts all        : use the -i option to state the inventory file
ansible -m ping all                               : ping all hosts
ansible -m command -a "hostname" all              : get the hostname of all inventory machines. -a "hostname" is the argument
ansible -m setup <NODE>                           : gather all the fact for a node
ansible-playbook <playbook_file.yml>              : run a playbook
ansible-galaxy init <ROLE_NAME>                   : creates a role directory structure from the current directory
ansible-vault create <FILE_NAME>                  : creates a new encrypted vault file in the current directory, will ask
                                                    for password for new file then jump you into an editor to add values (might need to export the EDITOR env to vi e.g. export EDITOR=vi)
ansible-vault edit <FILE_NAME>                    : edit the vault file
ansible-playbook ... --ask-vault-pass=...         : when running a playbook that uses a vault, use the option to provide
                                                    the password
ansible-playbook site.yml --list-tags             : find all the tags assigned to tasks in this playbook
ansible-playbool site.yml --tags "install"        : run the playbook but only the tasks with the install tag
ansible-playbool site.yml --skip-tags "install"   : run the playbook and run all but the install task



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
- They run concurrently across all selected nodes for performance reasons
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
at the play level set to true
- When using become, you may need to provide the sudo password, use the ansible-playbook option --ask-become-pass to get
ansible to request the password before running
```
---
  - hosts: all # the first play
    become: true
    tasks:
```
- If you have a task that is very similar and repeated you can do the following instead of copying and pasting the task
- Its possible to loop over lists for a single task by using the with_items syntax. Which uses the jinja (python) syntax
- Here the item variable will be replaced by each value within the with_items list
```
---
- hosts: db
  become: true
  tasks:
    - name: install random packages
      apt: name={{item}} state=present
      with_items:
        - apache2
        - mysql
        - nginx
```

### Services
- After installing packages you will need to tell Ansible about them so that it can manage them, You do that by using the
  service module

  ```
  ---
  - hosts: db
    become: true
    tasks:
      - name: install random packages
        apt: name=blah state=present
      - name: ensure the blah service is up and running
        service:
          name: blah
          state: started #this could be started/stopped/restarted/reloaded
          enabled: yes # or no - should the service be started when the
  ```
- after installing a package and making modifications to it, you would probably need to restart the service to get the changes
  enabled. So adding a new task that restarts the service isn't the way to go because if you were to run the playbook again,
  Ansible wouldn't do anything in terms of configuration changes but then would restart again if there was a restart task.
  - To make sure that restarts only happen when there are configuration changes we need to use Handlers

### Handlers
- Handlers in a playbook will never run unless instructed to by using notify on a task.
- Notify only gets triggered when the task has executed and resulted in a change
- If there are multiple calls to handlers during a play, they will dedupe and only run once
```
---
- hosts: db
  become: true
  tasks:
    - name: install apache2
      apt: name=apache2 state=present

    - name: ensure the apache2 service is up and running
      service:
        name: apache2
        state: restarted

    - name: enable an apache module
      apache2_module:
        name: mod_jk
        state: present
      notify: restart apache2

  handlers:
    name: restart apache2
    service:
      name: apache2
      state: restarted
```

### Copying files
- typically customisation involves copying files, to do so, use the copy module
```
---
- hosts: db
  become: true
  tasks:
   .....
    - name: copy files
      copy:
        src: ./playbooks/files/foo.conf #source is from the directory we ran the ansible-playbook command
        dest: /etc/foo.conf
        owner: foo
        group: foo
        mode: 0644
      notify: restart apache2

  handlers:
    name: restart apache2
    service:
      name: apache2
      state: restarted
```

### Editing files
- to edit a file in the target machine, use the file module.

```
---
- hosts: db
  become: true
  tasks:
   .....
    - name: remove default site from apache
      file:
        path: /etc/apache2/sites-enabled/000-default.conf
        state: absent
      notify: restart apache2
```

### Templates
- templates are files with jinja syntax in them that gets copied from the control/master machine to the target machine
  and have all their variables replaced
- template files end with the jinja 2 syntax "xxx.j2"
- have a look at the templates directory for the nginx.conf template
```
---
- hosts: nginx
  become: true
  tasks:
   .....
    - name: config nginx
      template:
        src: ./templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/n
        state: present
      notify: restart apache2
```

### line in files
- Ansilble gives users the ability to edit files but changing just one line or a group of them
  so that you do not need to copy and maintain a whole file locally or a template
- Ansible's answer to this is the lineInFiles module

```
---
- hosts: db
  become: true
  tasks:
   .....
    - name: enable db to bind and listen to all ip's
      lineinfile:
        path: /etc/mysql/my.cnf
        regexp: ^bind-address           #find the line in the file that starts with bind-address
        line: bind-address = 0.0.0.0
      notify: restart mysql

  handlers:
    - name: restart mysql
      service:
        - name: mysql
          state: restarted
```

### mysql
- you can create database users by using the mysql_user module
```
---
- hosts: db
  become: true
  tasks:
   .....
    - name: add new user
      mysql_user:
        name: paul
        password: paul
        priv: *.*:ALL
        host: localhost
```

### wait_for
- You can use the command module with 'service xxx status' to check that services are currently running
- You can also use the wait_for module to wait and test for a condition such as if the service is running on the right port
```
---
- hosts: db
  become: true
  tasks:
   .....
    - name: verify mysql running on 3306
      wait_for:
        port: 3306
```

### Register
- When running a task, you can get the result of that task and place it within a variable with the register module
```
---
- hosts: lb
  become: true
  tasks:
   .....
    - name: verify e2e test
      uri:
        url: http://{{item}}
        return_content: yes
        with_items: groups.lb
      register: variable_name

    - name: fail when no content returned
      when: "'some expected text in response' no in item.content"  #look for certain text in the variable defined in the register
      with_items: {{variable_name.results}}
```

### Roles
- Our playbooks as they exist at the moment configure machines by their type (layer in the architecture db/webserver/lb)
- If we wanted to share them as they are, they may need to be customised for different teams/use cases
- So copies will happen and maintaince will be hell
- To fix this, we will use roles, where the main shape of the stack is shared but there will be points where it is
  made to be customised
- To start a role, use the ansible-galaxy init <ROLE_NAME> command, this will create the directory structure of a role
    - ansible-galaxy init load-balancer
- In the directory the will be dirs: defaults, files, handlers, meta, tasks, templates and vars
- To migrate a standard playbook you would:

#### Move Tasks
- Move the tasks out of the playbook into the main.yml file in the task directory
```
---
- hosts: lb
  tasks:
    .....     # cut all tasks defined here to the main.yml task directory
  handlers:
    .....     # cut all the handlers into the handlers main.yml handlers directory
```

- in the main.yml file
```main.yml in tasks directory
---              # the main.yml contains a list of tasks
- name: install tools
  apt:
    ....
  notify: restart nginx

- name: start service
  service:
    ...
```
```main.yml in the handlers directory
---
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

- Then in the original playbook you reference the new task by adding the roles property pointing to the role name
```
---
- hosts: lb
  roles:
    - load-balancer
```
- When using roles, any templates your referencing in your tasks will automatically look into the templates role directory
  so you just need to make sure the templates are in the role/templates directory and reference it directly without the directory name

```main.yml in role task
---
- name: do come config stuff
  template: src=my.cnf dest=...
```

### Site wide playbooks
- The site.yml playbook is a playbook of playbooks.
- uses the include property to reference other playbooks and run them

```site.yml
---
- include: lb.yml
- include: database.yml
- include: website.yml
```

### Facts
- facts can be used to replace hard coded values in tasks using the jinja syntax

```main.yml in tasks dir
---
- name: update config
  lineInFile:
    path: /etc/mysql/my.cnf
    regexp: ^bind-address           #find the line in the file that starts with bind-address
    line: bind-address = {{ ansible_eth0.ipv4.address }}
```

- its also a good idea not to hard code values that can change between usages of roles so its a good idea
  to use variables and default values
- defaults can live in the main.yml in the defaults directory
- variable substitution is done using jinja syntax

```task.yml
---
- name:
  lineInFile:
    path: /a config file
    line: username = {{ db_username }} password = {{ db_password }}
```

```
---main.yml in defaults directory
db_username: defaultUser
db_password: defaultPassword
```
- You shouldn't just run playbooks with default values though, they should be replaced with the intended values
- overriding defaults are done by using variables
  - variables can be placed in multiple places and has a precidence list.
  - look up the documentation on its precedence
- its generally advised to keep variables simple and have perhaps 2 places where they should be defined
  - at the default level
  - at the playbook level with a map (hash data structure)
  - or with variable files
  - choose 2 and stick to them

```playbook.yml hash/map example
---
- host: lb
  become: true
  roles:
    - { role: loadbalancer, db_username: myCustomisedUsername, db_password: verySecretPassword }
```

- your able to use complex map/json/hash/dictionary structures within the jinja syntax when you use the with_dict key

```main.yml default directory
---
user:
  address:
    line1:
    line2:
```

```main.yml in the task directory
---
- name: update config
  template:
    src: myConfigTemplate.cnf
    dest: /etc/someApp/{{ item.key }}/config.cnf   ### here the item.key will reference the default value 'address' as were not being specific
  with_dict: user         #### HERE YOU REFERENCE THE TOP LEVEL KEY IN THE DEFAULT VARS
```
- you can also use the dict objects within templates
``` template.yml
upstream {{ item.key }} {

{ % for server in groups.nodes % }
  server {{ server }}: {{ item.value.backend }}
{ % endfor % }

}

server {
  listen {{ item.value.frontend }};

  location / {
   proxy_pass http://{{ item.key }};
  }

}

```

### Selective removals
- after running a successful playbook, the state of the target system as been changed
- e.g. new users created etc
- if you were to edit the tasks now to create a user with a different name, it would probably run BUT
  the old user will still exist.
- one way to deal with this is to create a clean up task that will remove all old stuff before creating the new stuff
- effectively starting from a clean state
- the way to do this is to use a number of modules, shell (gives a full shell), register and with_items

```main.yml in task
---
- name: clean up
  shell: ls -1 /etc/apache2/sites-enabled
  register: active_sites

- name: remove sites
  file:
    src: /etc/apache/sites-enabled/{{ item }}
    state: absent
  with_items: active_site.stout_lines
  when: item not in sites     ### references the sites key in the default vars only run when the item doesn't exist

```

### Group variables
- sometimes when working on a project, you dont have control of all your variable names
- sometimes you have multiple variable names with the same value
- when updating the value, you'll end up updating/maintaining multiple variables
- to make things easier, you can link these variables together and make a file containing the source of truth
- this will allow you to make a change in one place but continue to use those variables
  - you have 3 options to do this
    - vars file (requires you to include this file anywhere you need it)
    - inventory file (not recommended as it should just contain the list of hosts and no logic)
    - group vars / group vars all (recommended its automatically imported)
- to use group vars
  - create a group_vars directory from your root dir
  - create a new all.yml file. because its named all, all groups will use it
  - reference the keys in the all.yml in each of the group playbooks

```all.yml in the group_vars directory
db_user: aUserName
db_name: theDBName
db_pass: pass
```

```playbook.yml
---
- host: lb
  roles:
    - role: load-balancer
      db_user_name: {{ db_user }}   ## HERE THE db_user_name variable is sent to the load-balancer task but it
                                    ## refers to the db_user variable defined in the all.yml file
```

### Vault
- used to store sensitive data/variables like passwords
- theres a recommended standard to encrypt the file without losing what variables/groups the encrypted file contains
- the directory structure changes for group vars in that it should now look like /group_vars/<GROUP>/vars
- in the group dir use the ansible-vault create <FILE_NAME> command where you'll get a change to create a new file
- the new file will be in yml format so needs to start with --- at the top

```vault.yml
---
vault_db_user: dfsdfsd
vault_db_pass: sdfdsfsdf
```
- in the vars file now in the <GROUP> dir refer to the vault keys
```vars.yml
---
db_user: "{{ vault_db_user }}"
db_pass: "{{ vault_db_pass }}"
```
- to use a playbook that uses a vault you'll need to provide it with the vault password
  - ansible-playbook ... --ask-vault-pass=...
  - or you can set the password in a config file in you home diretory  ~/.vault_pass.txt
  - chmod 0600 ~/.vault_pass.txt
  - then update the ansilble.cfg file in the root and set the property vault_password_file=~/.vault_pass.txt

### Galaxy
- galaxy.ansible.com
- just like the chef scripts etc, where people share their playbooks and roles
- ansible-galaxy install <ROLE_NAME>


### improvements
- gather facts can take a while, you dont always need them do you can disable them using the 'gather_facts: false' key in the playbook
- multiple apt usages with update cache will run an apt update all the time even if you just ran it, you can skip it by using the cache_valid_time=86400 which caches the result for the next 24 hours (its in seconds)
- another way to improve apt is to have a playbook hit all target nodes with just an apt update with the cache setting before running any other playbook. This will allow you to parallize the apt update on all hosts at once. This will allow you to then remove the update cache from all apt modules in your tasks

```site.yml playbook. all playbook
---
  - hosts: all
    become: true
    gather_facts: false
    tasks:
      - name: update apt
        apt: update_cache=yes cache_valid_time=86400
- include: lb.yml
- include: db.yml
- include: web.yml
```

- You can limit the hosts targeted by using the --limit option. this works well if you wanted to run an entire site.yml but only wanted to target a single host.
- You can also limit tasks that run by applying tags to tasks and running the playbook with --tags "xxx"

```main.yml in the role tasks directory
---
 - name: install tool
   apt: ...
   tags: [ `install` ] ## provide a list of tags, you can also you the other list syntax with '-'
```





 =
