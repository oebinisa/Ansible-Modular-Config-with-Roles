# Ansible-Modular-Config-with-Roles

## Introduction

This is Phase 2 for the Ansible-Config repository. It refactors Ansible configuration modularly using roles and variables.

Basic Ansible configuration to set up a 3-tier web application architecture consisting of a load balancer tier (NGiNX), an applicaation tier - two Web servers (Apache webserver), and a database tier (MySQL server). All being accessed via a control machine/system.

- All commands to be run through a role in the Control machine.
- The Control machine needs SSH access to the other machines

## System Structure Setup:

                ─────────
               │Control
               └─────────
     │──────────────│────────────────│
     │              │                │
    ──────      ──────────────      ──────
    │lb01       │App01 - App02      │db01
    └──────     └──────────────     └──────
    LoadBalancer    Webserver       Database

## Steps

1.  Install Ansible on the Control Machine:

    Web: https://docs.ansible.com/

    Prerequisite: Python 2.6 or 2.7

    OS: Fresh Ubuntu Installation

            sudo apt-get install software-properties-common
            sudo apt-add-repository ppa:ansible/ansible
            sudo apt-get update
            sudo apt-get install ansible

    OS: Mac

            pip install --user ansible (or)
            brew install ansible

    Verify Installation:

            ansible-playbook --version

2.  Create a Project folder.

    Create the roles folder inside the project folder

        mkdir roles
        cd roles

    Create the skeletal folder structure for roles that mimics your server structure: Control, nginx, apache2, and mysql including a demo_app where your application would be copied into, using the commands below inside the newly created roles folder:

        ansible-galaxy init control
        ansible-galaxy init nginx
        ansible-galaxy init apache2
        ansible-galaxy init mysql
        ansible-galaxy init demo_app

    Concluding folder content/structure:

        .
        ├── group_vars
        │   └── all
        │       ├── vars.yml
        │       └── vault
        ├── playbooks
        │   ├── hostname.yml
        │   ├── stack_restart.yml
        │   └── stack_status.yml
        ├── roles
        │   ├── apache2
        │   ├── control
        │   │   ├── defaults
        │   │   │   └── main.yml
        │   │   ├── files
        │   │   ├── handlers        # All handler codes for control go here
        │   │   │   └── main.yml
        │   │   ├── meta
        │   │   │   └── main.yml
        │   │   ├── tasks           # All task codes for control go here
        │   │   │   └── main.yml
        │   │   ├── templates
        │   │   ├── tests
        │   │   │   ├── inventory
        │   │   │   └── test.yml
        │   │   └── vars
        │   │       └── main.yml
        │   ├── demo_app
        │   ├── mysql
        │   └── nginx
        ├── ansible.cfg             # Helps to parse the hosts into Ansible
        ├── control.yml             #
        ├── database.yml            #
        ├── inventory.txt           # Specify potential target machines in advance
        ├── loadbalancer.yml        #
        ├── site.yml                #
        └── webserver.yml           #

3.  Create the inventory.txt file (note the groupings)

        [loadbalancer]
        lb01

        [webserver]
        app01
        app02

        [database]
        db01

        [control]
        controlhost ansible_connection=local

    Verify: (inside project folder)

        ansible -i inventory.txt --list-hosts all

4.  Create ansible.cfg file:

    This helps to parse the hosts into Ansible without mentioning the local inventory

        [default]
        inventory = ./inventory.txt
        vault_password_file = ~/.vault_pass.txt

        #[ssh_connection]
        #pipelining = true

    Verify:

        ansible --list-hosts all

5.  Create Playbooks and setup the servers.

    Ensure all necessary packages are installed on the various machines. Install and configure them accordingly

    **5a. Hostnames**

    Create main playbooks/hostname.yml:

        ---
        - hosts: all
          tasks:
            - name: get server hostname
              command: hostname

        # See playbooks/hostname.yml for complete code

    **5b. Control:**

    This would help to update and install all dependencies on the Control Machine

    Create control.yml:

        ---
        - hosts: control
          become: true
          roles:
            - control   # Points to the specific folder to look into for commands

        # See control.yml for complete code

    **Control Tasks:**

    Update roles/control/tasks/main.yml with the necessary tasks

        ---
        - name: install tools
          apt: name={{item}} state=present
          with_items:
            - curl
            - python-httplib2

        # See roles/control/tasks/main.yml for complete code

    **5c. Load Balancer (nginx):**

    Create loadbalancer.yml:

        ---
        - hosts: loadbalancer
          become: true
          roles:
            - nginx

        # See loadbalancer.yml for complete code

    **LB Tasks:**

    Update roles/nginx/tasks/main.yml with the necessary tasks

        ---
        - name: install tools
          apt: name={{item}} state=present
          with_items:
            - python-httplib2

        - name: install nginx
          apt: name=nginx state=present

        # See roles/nginx/tasks/main.yml for complete code

    **LB Handlers**:

    Update roles/nginx/handlers/main.yml with the necessary handler codes

        ---
        - name: restart nginx
          service: name=nginx state=restarted

        # See roles/nginx/handlers/main.yml for complete code

    **LB Defaults**:

    Update roles/nginx/defaults/main.yml with the necessary default values:

        ---
        sites:
          myapp:
            frontend: 80
            backend: 80

        # See roles/nginx/handlers/main.yml for complete code

    **LB Templates**:

    Configure the Load Balancer (NGiNX) template to render pages from the Webservers instead of the default NGiNX pages using the NGiNX template file module

    Instead of just copying like the copy module, the template would sub out any variables supplied first before rendering the final content into the end host.

    Update roles/nginx/templates/nginx.conf.j2 as follows:

        upstream {{ item.key }} {
            {% for server in groups.webserver %}
                server {{ server }}:{{ item.value.backend }};
            {% endfor %}
        }

        server {
            listen {{ item.value.frontend }};

            location / {
                proxy_pass http://{{ item.key }};
            }
        }

    **5d: Webserver (apache):**

    Create webserver.yml:

        ---
        - hosts: webserver
          become: true
          roles:
            - apache2
            - demo_app

        # See webserver.yml for complete code

    **Webserver Tasks:**

    Update roles/apache2/tasks/main.yml with the necessary tasks.

        ---
        - hosts: webserver
          become: true
          tasks:
            - name: install web components
              apt: name={{item}} state=present
              with_items: # update apache dependencies on the webserver
                - apache2
                - libapache2-mod-wsgi

        # See roles/apache2/tasks/main.yml for complete code

    **Webserver Handlers**:

    Update roles/apache2/handlers/main.yml with the necessary handler codes

        ---
        - name: restart apache2
          service: name=apache2 state=restarted

        # See roles/apache2/handlers/main.yml for complete code

    **App Tasks:**

    Update roles/demo_app/tasks/main.yml with the necessary tasks

    This go into the demo_app tasks folder:

        ---
        - hosts: webserver
          become: true
          tasks:
            - name: install web components
              apt: name={{item}} state=present
              with_items: # update python dependencies
                - python-pip
                - python-virtualenv
                - python-mysqldb

        # See roles/demo-app/tasks/main.yml for complete code

    **App Handlers:**

    Update roles/demo_app/handlers/main.yml with the necessary handler codes

        ---
        - name: restart apache2
          service: name=apache2 state=restarted

        # See roles/demo_app/handlers/main.yml for complete code

    **App Templates:**

    Update the App role template file with vaariables for the Database connection.

    Update roles/demo_app/templates/demo.wsgi.j2 as follows:

        activate_this = '/var/www/demo/.venv/bin/activate_this.py'
        execfile(activate_this, dict(__file__=activate_this))

        import os
        os.environ['DATABASE_URI'] = 'mysql://{{ db_user }}:{{ db_pass }}@{{ groups.database[0] }}/{{ db_name }}'

        import sys
        sys.path.insert(0, '/var/www/demo')

        from demo import app as application

    **_Note:_**
    Move your entire demo app folder into demo_app/files folder

    **5e. Database (mysql):**

    Create database.yml:

        ---
        - hosts: database
          become: true
          roles:
            - role: mysql
                db_user_name: "{{ db_user }}"
                db_user_pass: "{{ db_pass }}"
                db_user_host: "%"


        # See database.yml for complete code

    **DB Tasks:**

    Update roles/mysql/tasks/main.yml with the necessary tasks

        ---
        - name: install tools
          apt: name={{item}} state=present
          with_items:
            - python-mysqldb

        - name: install mysql-server
          apt: name=mysql-server state=present

        # See roles/mysql/tasks/main.yml for complete code

    **DB Handlers:**

    Update roles/mysql/handlers/main.yml with the necessary tasks

        ---
        - name: restart mysql
          service: name=mysql state=restarted

        # See roles/mysql/handlers/main.yml for complete code

    **DB Defaults:**

    Create variables that are referenced in database.yml and roles/mysql/tasks/main.yml

    Update roles/mysql/defaults/main.yml

        ---
        db_name: myapp
        db_user_name: dbuser
        db_user_pass: dbpass
        db_user_host: localhost

6.  Aggregate and Encrypt Variables

    **Aggregate Variables:**

    Aggregate all variables into a group file. It provides a one-stop-shop where you can update your variables across board.

    Create the following in the project directory: group_vars/all/vars.yml

        ---
        db_name: demo
        db_user: demo
        db_pass: "{{ vault_db_pass }}"

    **Encrypt Your Aggregated Variables:**

    Ansible Vault comes in handy in encrypting aggregated variables.

        cd group_vars/all
        export EDITOR=vi
        ansible-vault create vault

    When promted, enter a passphrase that will be used to encrypt the file (twice).

    Enter the following code and save.

        ---
        vault_db_pass: myvaultpass

    To edit the vault file, use the following:

        ansible-vault edit vault

    **Parsing Vault Passphrase into Ansible:**

    Ansible would need to access the Vault passphrase during execution.

    Therefore, create a secure file in your home directory and pass it into Ansible:

        echo "myvaultpass" > ~/.vault_pass.txt
        chmod 0600 !$

    Ensure that there is a corresponding entry to your new vault password file in the ansible.cfg file

        vault_password_file = ~/.vault_pass.txt

7.  Create playbook that aggregates and executes all the above playbooks in bulk

    Create site.yml:

        ---
        - hosts: all
          become: true
          gather_facts: false
          tasks:
            - name: update apt cache
              apt: update_cache=yes cache_valid_time=86400

        - include: playbooks/hostname.yml
        - include: control.yml
        - include: loadbalancer.yml
        - include: webserver.yml
        - include: database.yml

8.  Create Operational Playbooks: stack_restart.yml and stack_status.yml

    These would help the ease of restarting the stack and checking stack status post implementation/configuration

    Create playbooks/stack_restart.yml:

        ---
        # Bring stack down
        - hosts: loadbalancer
          become: true
          tasks:
            - services: name=nginx statee=stopped
            - wait_for: port=80 state=drained

        - hosts: webserver

        # See playbooks/stack_restart.yml for complete code

    Create playbooks/stack_status.yml

        ---
        - hosts: loadbalancer
          become: true
          gather_facts: false
          tasks:
            - name: verify nginx service
              command: service nginx status
              changed_when: false

            - name: verify nginx is listening on 80
              wait_for: port=80 timeout=1 # default is 300secs

        # See playbooks/stack_status.yml for complete code

## Other Tasks:

======

Ping All:

    ansible -m ping all     # Test Ansible connectivity to all hosts

    ansible -m command -a "hostname" all
    ansible -a "hostname" all

    ansible -a "/bin/false" all

The last 2nd and 3rd are same cos command is the default module.

Get more module commands at https://doc.ansible.com/ansible/modules_by_category.html

**Curl:**

Curl the Webservers, the Load Balancer, and the Database a couple of times to verify connection

    curl app01
    curl app02

    curl lb01
    curl lb01
    curl lb01

    curl app01/db
    curl app02/db
    curl lb01/db
    curl lb01/db
    curl lb01/db

**Limiting Execution by Hosts:**

site.yml contains aggregated playbooks. All the servers and services are executed when you call this script.

Use the _limit_ option to limit the execution a few or a single/specific server i.e.

    ansible-playbook site.yml --limit app01

**Limiting Execution by Tasks (using Tags):**

To select specific task(s) and apply to our hosts, use tags

Lists all tags mentioned in the playbooks:

    ansible-playbook site.yml --list-tags

To execute a particula tag:

    ansible-playbook site.yml --tags "packages"

Execute everything except a particula tag:

    ansible-playbook site.yml --skip-tags "packages"
