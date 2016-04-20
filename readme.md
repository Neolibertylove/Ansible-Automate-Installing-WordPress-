How To Automate Installing WordPress on Linux Using Ansible
================

Step 1 — Installing Ansible:

sudo apt-get install ansible

Show the version of Ansible 

ansible –version

Step 2 — Setting Up the File Structure:

Create a directory for our playbook:
cd ~
mkdir wordpress-ansible && cd wordpress-ansible

into this directory and create two files: one called playbook.yml
and another called hosts

touch playbook.yml
touch hosts

mkdir roles && cd roles

For each role that we want to create, we will run ansible-galaxy init:

ansible-galaxy init server
ansible-galaxy init php
ansible-galaxy init mysql
ansible-galaxy init wordpress

Step 3 - Writing the Playbook:
Edit hosts:
nano ~/wordpress-ansible/hosts

[wordpress]
localhost

Edit the playbook file:

nano ~/wordpress-ansible/playbook.yml

- hosts: wordpress
  roles:
    - server
    - php
    - mysql
    - wordpress
cd ~/wordpress-ansible/

Step 4- Creating Roles:

The server role will install all the software we need

nano roles/server/tasks/main.yml

- name: Update apt cache
  apt: update_cache=yes cache_valid_time=3600
  sudo: yes

- name: Install required software

  apt: name={{ item }} state=present
  sudo: yes
  with_items:
    - apache2
    - mysql-server
    - php5-mysql
    - php5
    - libapache2-mod-php5
    - php5-mcrypt
    - python-mysqldb
	
Edit the main tasks file for PHP:

nano roles/php/tasks/main.yml

- name: Install php extensions
  apt: name={{ item }} state=present
  sudo: yes
  with_items:
    - php5-gd 
    - libssh2-php

set up a MySQL database for our WordPress site :

nano roles/mysql/defaults/main.yml

wp_mysql_db: wordpress
wp_mysql_user: wordpress
wp_mysql_password: wp_db_password

nano roles/mysql/tasks/main.yml

- name: Create mysql database
  mysql_db: name={{ wp_mysql_db }} state=present

- name: Create mysql user
  mysql_user: 
    name={{ wp_mysql_user }} 
    password={{ wp_mysql_password }} 
    priv=*.*:ALL

we can set up WordPress. We'll be editing the wordpress role :

nano roles/wordpress/tasks/main.yml



- name: Download WordPress  get_url: 
    url=https://wordpress.org/latest.tar.gz 
    dest=/tmp/wordpress.tar.gz
    validate_certs=no

- name: Extract WordPress  unarchive: src=/tmp/wordpress.tar.gz dest=/var/www/ copy=no
  sudo: yes

- name: Update default Apache site
  sudo: yes
  lineinfile: 
    dest=/etc/apache2/sites-enabled/000-default.conf 
    regexp="(.)+DocumentRoot /var/www/html"
    line="DocumentRoot /var/www/wordpress"
  notify:
    - restart apache

- name: Copy sample config file
  command: mv /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php creates=/var/www/wordpress/wp-config.php
  sudo: yes

- name: Update WordPress config file
  lineinfile:
    dest=/var/www/wordpress/wp-config.php
    regexp="{{ item.regexp }}"
    line="{{ item.line }}"
  with_items:
    - {'regexp': "define\\('DB_NAME', '(.)+'\\);", 'line': "define('DB_NAME', '{{wp_mysql_db}}');"}        
    - {'regexp': "define\\('DB_USER', '(.)+'\\);", 'line': "define('DB_USER', '{{wp_mysql_user}}');"}        
    - {'regexp': "define\\('DB_PASSWORD', '(.)+'\\);", 'line': "define('DB_PASSWORD', '{{wp_mysql_password}}');"}
  sudo: yes

nano roles/wordpress/handlers/main.yml

- name: restart apache
  service: name=apache2 state=restarted
  sudo: yes      

run and will install automatic 

ansible-playbook playbook.yml -i hosts -u
