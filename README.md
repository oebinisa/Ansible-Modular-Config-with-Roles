# Ansible-Modular-Config-with-Roles

## Introduction

This is Phase 2 for the Ansible-Config repository. It refactors Ansible configuration modularly using roles.

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

    Create the skeletal folder structure for Control, nginx, apache2, the demo app and mysql using the commands below inside the newly created roles folder:

        ansible-galaxy init control
        ansible-galaxy init nginx
        ansible-galaxy init apache2
        ansible-galaxy init demo_app
        ansible-galaxy init mysql

    Your resulting folder would look like below:

        .
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

    Concluding folder content/structure:

        .
        ├── inventory.txt           # Specify potential target machines in advance
        ├── ansible.cfg             # Helps to parse the hosts into Ansible
        ├── playbooks               # Plays are a group of tasks that can be executed in bulk
        │   ├── hostname.yml
        │   ├── stack_restart.yml
        │   └── stack_status.yml
        ├── control.yml             #
        ├── database.yml            #
        ├── database.yml            #
        ├── loadbalancer.yml        #
        ├── webserver.yml           #
        └── templates
            └── nginx.conf.j2

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

    Verify:

        ansible --list-hosts all

5.  Create Playbooks.

    Ensure all necessary packages are installed on the various machines.

    Install and configure them accordingly

5a. Create main playbooks/hostname.yml:

        ---
        - hosts: all
          tasks:
            - name: get server hostname
              command: hostname

    # See playbooks/hostname.yml for complete code

5b. Control

    This would help to update and install all dependencies on the Control Machine

    Create control.yml:

            ---
            - hosts: control
            become: true
            roles:
              - control   # Points to the specific folder to look into for commands

    # See control.yml for complete code

    Update roles/control/tasks/main.yml with the necessary tasks

            ---
            - name: install tools
            apt: name={{item}} state=present update_cache=yes
            with_items:
              - curl
              - python-httplib2

        # See roles/control/tasks/main.yml for complete code

    Create loadbalancer.yml:

        ---
        - hosts: loadbalancer
          become: true
          tasks:
            - name: install nginx
            apt: name=nginx state=present update_cache=yes

        # See loadbalancer.yml for complete code

    Create webserver.yml:

        ---
        - hosts: webserver
          become: true
          tasks:
            - name: install web components
              apt: name={{item}} state=present update_cache=yes
              with_items:
                - apache2
                - libapache2-mod-wsgi
                - python-pip
                - python-virtualenv
                - python-mysqldb

        # See webserver.yml for complete code

    Create database.yml:

        ---
        - hosts: database
          become: true
          tasks:
            - name: install mysql-server
            apt: name=mysql-server state=present update_cache=yes

        # See database.yml for complete code

Executive the above using the script below:

        ansible-playbook playbooks/hostname.yml
        ansible-playbook control.yml
        ansible-playbook loadbalancer.yml
        ansible-playbook webserver.yml
        ansible-playbook database.yml

6.  Configure the Load Balancer (NGiNX) to render pages from the Webservers instead of the default NGiNX pages using the NGiNX template file module

    Instead of just copying like the copy module, the template would sub out any variables supplied first before rendering the final content into the end host.

    Create templates/nginx.conf.j2:

        upstream demo {
            {% for server in groups.webserver %}
                server {{ server }};
            {% endfor %}
        }

        server {
            listen 80;

            location / {
                proxy_pass http://demo;
            }
        }

    Execute the loadbalancer playbook when done:

        ansible-playbook loadbalancer.yml

7.  Create Operational Playbooks: stack_restart.yml and stack_status.yml

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
          tasks:
            - name: verify nginx service
              command: service nginx status

            - name: verify nginx is listening on 80
              wait_for: port=80 timeout=1 # default is 300secs

        # See playbooks/stack_status.yml for complete code

## Tasks:

======

Ping All:

    ansible -m ping all     # Test Ansible connectivity to all hosts

    ansible -m command -a "hostname" all
    ansible -a "hostname" all

    ansible -a "/bin/false" all

The last 2nd and 3rd are same cos command is the default module.

Get more module commands at https://doc.ansible.com/ansible/modules_by_category.html

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
