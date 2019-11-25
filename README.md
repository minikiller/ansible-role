# ansible-role

Table of Contents
Roles Lab
1. Connect to Environment
2. Create Roles
2.1. Create Role to Set Up Web Services
2.2. Create Role to Set Up Database
2.3. Create Role to Set Up Load Balancer
3. Create and Execute Main Playbook
4. Test Playbook
5. Evaluate Your Progress
Roles Lab
In this lab, you create Ansible roles that use variables, files, templates, tasks, and handlers to deploy a network service and enable a working firewall. You then use Ansible Galaxy to initialize a new Ansible role, and download and install an existing role.

Goals
Create Ansible roles to deploy a network service and enable a working firewall

Use Ansible Galaxy to initialize, download, and install roles

1. Connect to Environment
Set some useful environment variables:

[laptop ]$ export GUID=<"GUID from email">
[laptop ]$ export MYKEY=<~/.ssh/your_key.pem>
[laptop ]$ export MYUSER=<username-company.com>
Example
[laptop ]$ export GUID=e4gh
[laptop ]$ export MYKEY=~/.ssh/psrivatkey
[laptop ]$ export MYUSER=psrivast-redhat.com
Connect to the bastion host with your OPENTLC ID and private key:

[laptop ]$ ssh -i ${MYKEY} ${MYUSER}@bastion.${GUID}.example.opentlc.com
Log in as the devops user:

[user-company.com@bastion ~]$ sudo -i
[root@bastion ~]# su - devops
Run the lab-5.1-setup.yml playbook to set up the lab environment:

[devops@bastion ~]$ cd ~/ansible_implementation_grading/
[devops@bastion ansible_implementation_grading]$ ansible-playbook lab-5.1-setup.yml
Change to the ansible_implementation directory:

[devops@bastion ansible_implementation_grading]$ cd ~/ansible_implementation
2. Create Roles
In this section, you create roles to deploy the web application. You create a role to set up the Apache web server. Then you create a role to install mariadb and use a database backup file to populate the database. Lastly, you create a role for setting up a HAProxy load balancer for high availability for your web application.

2.1. Create Role to Set Up Web Services
In this section, you create a role to set up httpd services.

Create a role called app-tier using the ansible-galaxy command.

Add a task to install and enable firewalld.

Add a task to install and start httpd.

Add a task to create a custom vhost.conf configuration file under the /etc/httpd/conf.d/ directory.

The vhost.conf.j2 template is already created to help you with this step.

Add a task to create the /var/www/vhost/ document root directory.

Add a task to create an index.php file in the document root directory using index.j2 as the template.

Add a task to open the firewall ports as per the requirements.

Enable SELinux so that the Apache back-end server can connect to the database:

httpd_can_network_connect_db

httpd_can_network_connect

Create the vars/main.yml file under the app-tier role directory that contains definitions for all of the variables defined in tasks.

Create the file handlers/main.yml under the app-tier role directory that contains a handler to restart services if needed.

Playbook Solution
[devops@bastion ansible_implementation]$ mkdir roles/
[devops@bastion ansible_implementation]$ ansible-galaxy init roles/app-tier
[devops@bastion ansible_implementation]$ cat << EOF > roles/app-tier/tasks/main.yml
---
# tasks file for roles/app-tier
# Installation of packages based on inventory groupss
- name: Install Firewalld
  yum:
   name: firewalld
   state: latest

- name: Start firewalld service
  service:
   name: firewalld
   state: started
   enabled: true

- name: Install httpd
  yum:
   name: "{{ item }}"
   state: latest
  with_items:
   - "{{ httpd_pkg }}"

- name: Start httpd
  service:
   name: "{{ httpd_srv }}"
   enabled: true
   state: started


- name: Copy vhost template file
  template:
   src: vhost.conf.j2
   dest: /etc/httpd/conf.d/vhost.conf
  notify:
   - restart_httpd

- name: Create Document Root
  file:
   path: /var/www/vhost/
   state: directory

- name: Copy index.j2 file
  template:
   src: index.j2
   dest: /var/www/vhost/index.php
   mode: 0644
   owner: apache
   group: apache

- name: Open httpd port
  firewalld:
   service: http
   state: enabled
   immediate: true
   permanent: true

- name: enable selinux boolean
  seboolean:
   name: "{{ item }}"
   state: yes
   persistent: yes
  loop:
   - httpd_can_network_connect_db
   - httpd_can_network_connect

EOF

[devops@bastion ansible_implementation]$ cat << EOF > roles/app-tier/handlers/main.yml
---
# handlers file for roles/app-tier

- name: restart_httpd
  service:
   name: "{{ httpd_srv }}"
   state: restarted

EOF

[devops@bastion ansible_implementation]$ cat << EOF > roles/app-tier/vars/main.yml
---
# vars file for roles/app-tier

db:
 user: root
 database: userdb
 password: redhat
httpd_pkg:
 - httpd
 - php
 - php-mysql
httpd_srv: httpd
db_srv: mariadb
EOF

[devops@bastion ansible_implementation]$ cp ~/roles-setup-files/index.j2 roles/app-tier/templates/
[devops@bastion ansible_implementation]$ cp ~/roles-setup-files/vhost.conf.j2 roles/app-tier/templates/
2.2. Create Role to Set Up Database
In this section, you create a role to install mariadb services and restore the backup file.

Create a role called db-tier using the ansible-galaxy command.

Add tasks to your playbook to install and enable the mariadb service, and start firewalld, in a similar manner as the previous exercise.

Open firewall ports as per the requirements.

Add tasks to check if the mariadb root password is set and set a password as specified in playbook variables.

Add a task to ensure that users have the appropriate privileges on the database.

Add a task to copy the userdb.backup database backup file to the server.

Add a task to restore the userdb.backup backup file for mariadb data.

Create a vars/main.yml file under the db-tier role that defines values for all of the variables defined in the tasks, including these values for the database:

user: root

password: redhat

database: userdb

backup file name: userdb.backup

Playbook Solution
[devops@bastion ansible_implementation]$ ansible-galaxy init roles/db-tier
[devops@bastion ansible_implementation]$ cp ~/roles-setup-files/userdb.backup roles/db-tier/files/
[devops@bastion ansible_implementation]$ cat << EOF > roles/db-tier/tasks/main.yml
---
# tasks file for roles/db-tier
- name: Install mysql
  yum:
   name: "{{ item  }}"
   state: latest
  loop:
   - "{{ db_pkg }}"

- name: Start mysql
  service:
   name: "{{ db_srv }}"
   enabled: true
   state: started

- name: Start firewalld
  service:
   name: firewalld
   state: started
   enabled: true

- name: Open mysql port
  firewalld:
   service: mysql
   state: enabled
   immediate: true
   permanent: true

- name: Check if root password is set
  shell: >
    mysqladmin -u root status
  changed_when: false
  failed_when: false
  register: root_pwd_check


- name: Setting up mariadb password
  mysql_user:
   name: "{{ db['user'] }}"
   password: "{{ db['password'] }}"
  when: root_pwd_check.rc == 0

- name: DB users have privileges on all databases
  mysql_user:
   name: "{{ db['user']}}"
   priv: "*.*:ALL"
   append_privs: yes
   password: "{{ db['password']}}"
   login_password: "{{ db['password']}}"
   host: "{{ item }}"
  loop:
   - "{{ inventory_hostname }}"
   - '%'

- name: Copy database dump file
  copy:
   src: "{{ db['backupfile']}}"
   dest: /tmp

- name: Restore database
  mysql_db:
   name: "{{ db['database'] }}"
   state: import
   target: "/tmp/{{ db['backupfile'] }}"
   login_password: "{{ db['password']}}"
EOF

[devops@bastion ansible_implementation]$ cat << EOF > roles/db-tier/vars/main.yml
---
# vars file for roles/db-tier
db_pkg:
 - mariadb
 - mariadb-server
 - MySQL-python
 - firewalld
db_srv: mariadb
db:
 user: root
 database: userdb
 password: redhat
 backupfile: userdb.backup
EOF
2.3. Create Role to Set Up Load Balancer
In this section, you create a role to install HAProxy services and use the webservers host group as the back end.

Create a role called lb-tier using the ansible-galaxy command.

Add tasks to install and start the firewall, then start HAProxy.

Add a task to copy an HAProxy template to the server, using the haproxy.j2 file as the template.

Add a task to open the required HAProxy ports.

Create a vars/main.yml file under the lb-tier role directory that contains definitions for the variables defined in the tasks.

Create the handlers/main.yml file under the lb-tier role directory that contains a handler to restart services if needed.

Playbook Solution
[devops@bastion ansible_implementation]$ ansible-galaxy init roles/lb-tier
[devops@bastion ansible_implementation]$ cp ~/roles-setup-files/haproxy.j2 roles/lb-tier/templates/
[devops@bastion ansible_implementation]$ cat << EOF > roles/lb-tier/tasks/main.yml
---
# tasks file for roles/lb-tier
- name: Install Firewalld
  yum:
   name: firewalld
   state: latest


- name: Start firewalld service
  service:
   name: firewalld
   state: started
   enabled: true

- name: Install haproxy
  yum:
   name: "{{ item  }}"
   state: latest
  loop:
   - "{{ haproxy_pkg }}"


- name: Start haproxy
  service:
   name: "{{ haproxy_srv }}"
   enabled: true
   state: started


- name: Copy haproxy template
  template:
   src: haproxy.j2
   dest: /etc/haproxy/haproxy.cfg
  notify:
   - restart_haproxy

- name: Open haproxy port
  firewalld:
   service: http
   state: enabled
   immediate: true
   permanent: true

- name: Open haproxy statistics port
  firewalld:
   port: 5000/tcp
   state: enabled
   immediate: true
   permanent: true
EOF


[devops@bastion ansible_implementation]$ cat << EOF > roles/lb-tier/handlers/main.yml
# handlers file for roles/lb-tier
- name: restart_haproxy
  service:
   name: "{{ haproxy_srv }}"
   enabled: true
   state: restarted
EOF

[devops@bastion ansible_implementation]$ cat << EOF > roles/lb-tier/vars/main.yml
---
# vars file for roles/lb-tier
haproxy_pkg:
 - haproxy
 - firewalld
haproxy_srv: haproxy
EOF
3. Create and Execute Main Playbook
In this section, you create and execute a main playbook to call all of the roles.

Create the main playbook to invoke the roles as follows:

Execute the lb-tier role on the lb host group servers.

Execute the db-tier role on the db host group servers.

Execute the app-tier role on the webservers host group servers.

Playbook Solution
[devops@bastion ansible_implementation]$ cat << EOF > webapp-main.yml
- hosts: webservers
  become: yes
  roles:
   - app-tier

- hosts: db
  become: yes
  roles:
   - db-tier

- hosts: lb
  become: yes
  roles:
   - lb-tier
EOF
Execute the main playbook:

[devops@bastion ansible_implementation]$ ansible-playbook webapp-main.yml
Sample Output
Output Ommitted....
PLAY RECAP ****************************************************************************************************************************************************
app1.3fd5.internal         : ok=10   changed=5    unreachable=0    failed=0
app2.3fd5.internal         : ok=10   changed=5    unreachable=0    failed=0
appdb1.3fd5.internal       : ok=9    changed=4    unreachable=0    failed=0
frontend1.3fd5.internal    : ok=9    changed=4    unreachable=0    failed=0
4. Test Playbook
Open a web browser window and enter the http://frontend1.${GUID}.example.opentlc.com/ URL.

When the web page prompts you for the username, enter kiosk.

Sample Output
kiosk redhat /bin/bash /home/kiosk
5. Evaluate Your Progress
Grade your work:

[devops@bastion ansible_implementation]$ cd ~/ansible_implementation_grading
[devops@bastion ansible_implementation_grading]$ export GUID=`hostname | awk -F"." '{print $2}'`
[devops@bastion ansible_implementation_grading]$ ansible-playbook lab-5.1-grade.yml -e GUID=${GUID}
Sample Output
Output Omitted...

TASK [Check that "roles" dir is present.] *********************************************************************************************************************
ok: [localhost]

TASK [Fail if "roles" directory is not present] ***************************************************************************************************************
skipping: [localhost]

TASK [Success if roles directory is present] ******************************************************************************************************************
ok: [localhost] => {
    "msg": "roles directory is present"
}

TASK [Check that "roles/lb-tier" dir is present.] *************************************************************************************************************
ok: [localhost]

TASK [Fail if "roles/lb-tier" directory is not present] *******************************************************************************************************
skipping: [localhost]

TASK [Success if roles directory is present] ******************************************************************************************************************
ok: [localhost] => {
    "msg": "roles/lb-tier directory is present"
}

TASK [Check that "roles/db-tier" dir is present.] *************************************************************************************************************
ok: [localhost]

TASK [Fail if "roles/db-tier" directory is not present] *******************************************************************************************************
skipping: [localhost]

TASK [Success if roles directory is present] ******************************************************************************************************************
ok: [localhost] => {
    "msg": "roles/db-tier directory is present"
}

TASK [Check that "roles/app-tier" dir is present.] ************************************************************************************************************
ok: [localhost]

TASK [Fail if "roles/app-tier" directory is not present] ******************************************************************************************************
skipping: [localhost]

TASK [Success if roles directory is present] ******************************************************************************************************************
ok: [localhost] => {
    "msg": "roles/app-tier directory is present"
}

TASK [Run a Curl Test against Frontend] ***********************************************************************************************************************
ok: [localhost] => (item=frontend1.3fd5.internal)

TASK [Fail if 'Enter User Name' is not in the page content] ***************************************************************************************************
skipping: [localhost]

TASK [Success if 'Enter User Name' is in the page content] ****************************************************************************************************
ok: [localhost] => {
    "msg": "Success: All requirements completed."

Output Omitted...
Correct any reported failures.

Rerun the script until you see no failures.


Build Version: 1.4R : Last updated 2019-02-22 12:33:16 EST
