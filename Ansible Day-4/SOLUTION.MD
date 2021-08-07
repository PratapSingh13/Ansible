# How to Use Ansible to Install and Set Up LAMP on Ubuntu 18.04


## Prerequisites
In order to execute the automated setup provided by the playbook we’re discussing in this guide, you’ll need:

**One Ansible control node:** An Ubuntu 18.04 machine with Ansible installed and configured to connect to your Ansible hosts using SSH keys. Make sure the control node has a regular user with sudo permissions and a firewall enabled, as explained in our Initial Server Setup guide. To set up Ansible, please follow our guide on How to Install and Configure Ansible on Ubuntu 18.04.

**One or more Ansible Hosts:** One or more remote Ubuntu 18.04 servers previously set up following the guide on How to Use Ansible to Automate Initial Server Setup on Ubuntu 18.04.

## Things to be installed on target machine
* For ‘portage’ support it requires the `qlist` utility, which is part of ‘app-portage/portage-utils’.
* For Debian-based systems `python-apt` package must be installed on targeted hosts.

> Before proceeding, you first need to make sure your Ansible control node is able to connect and execute commands on your Ansible host(s).

**For a connection test, please check step 3 of How to Install and Configure Ansible on Ubuntu 18.04.**
```sh
ansible -m ping all
```

It should be like this:
```json
node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}

node2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
 # What exactly this playbook does
 This Ansible playbook install and Set Up LAMP on Ubuntu 18.04.

 Running this playbook will perform the following actions on your Ansible hosts:

 - Install `aptitude`, which is preferred by Ansible as an alternative to the apt package manager.
 - Install Apache.
 - Create a custom document root folder for the new Apache VirtualHost and set up a test page.
 - Enable the new Apache VirtualHost.
 - Disable the default Apache website when the variable `disable_default` is set to `true`.
 - Set up UFW to allow HTTP traffic on the configured port (80 by default).
 
 Once the playbook has finished running, you will have a web server running on your target node, based on the options you defined within your configuration variables.

# How to Use this Playbook
The first thing we need to do is obtain the Apache playbook and its dependencies from the https://github.com/PratapSingh13/Ansible repository. We need to clone this repository to a local folder inside the Ansible Control Node.

```sh
cd ~
git clone https://github.com/do-community/ansible-playbooks.git
cd ansible-playbooks
```

The files we’re interested in are located inside the lamp-Ubuntu18.04 folder, which has the following structure:

```
lamp-Ubuntu18.04
|---files
|   |--apache.conf.j2
|   |--index.html.j2
|---vars
|   |--default.yml
|---playbook.yml
```

## Well what each of these files are? 
***files/apache.conf.j2-*** Template file for setting up the Apache Virtual Host.
***files/index.html.j2-*** Template file for setting up a test page on the web server’s root directory.
***vars/default.yml-*** Variable file for customizing playbook variables.
***playbook.yaml-*** The playbook file, containing the tasks to be executed on the remote server(s).

We’ll edit the playbook’s variable file to customize a few options. Access the lamp-Ubuntu18.04 directory and open the vars/default.yml file using your command line editor of choice:

`vars/default.yml`
```yaml
---
app_user: "user_name"
http_host: "your_domain"
http_conf: "your_domain.conf"
http_port: "80"
disable_default: true
```
Here is what each of these are:
***app_user-*** A remote non-root user on the Ansible host that will be set as the owner of the application files.
***http_host-*** Your domain name.
***http_conf-*** The name of the configuration file that will be created within Apache.
***http_port-*** HTTP port for this virtual host, where 80 is the default.
***disable_default-*** Whether or not to disable the default website that comes with Apache.

`files/apache.conf.j2`
```
<VirtualHost *:{{ http_port }}>
   ServerAdmin webmaster@localhost
   ServerName {{ http_host }}
   ServerAlias www.{{ http_host }}
   DocumentRoot /var/www/{{ http_host }}
   ErrorLog ${APACHE_LOG_DIR}/error.log
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
The `apache.conf.j2` file is a `Jinja 2` template file that configures a new Apache VirtualHost. The variables used within this template are defined in the `vars/default.yml` variable file.

`files/index.html.j2`
```
<html>
   <head>
       <title>Welcome to {{ http_host }} !</title>
   </head>
   <body>
       <h1>Success! The {{ http_host }} virtual host is working!</h1>
   </body>
</html>
```
The `index.html.j2` file is another Jinja template, used to set up a test HTML page in the document root of the newly configured Apache server.

Once you’re done updating the variables inside `vars/default.yml` save and close this file. You’re now ready to run this playbook on one or more servers. Most playbooks are configured to be executed on every server in your inventory, by default

`playbook.yml`
```yaml
---
- hosts: all
  become: yes
  vars_files:
    - vars/default.yaml
  tasks:
    - name: "Install prerequisites"
      apt: 
        name: '{{ item }}'
        update_cache: yes 
        state: latest 
        force_apt_get: yes
      loop: 
        - aptitude

  #Apache Configuration
    - name: Install LAMP Packages
      apt: 
        name: "{{ item }}" 
        update_cache: yes 
        state: latest
      loop:
        - apache2 
        - mysql-server 
        - python3-pymysql 
        - php 
        - php-mysql 
        - libapache2-mod-php
        - python3-pip
        - python3-mysqldb

    - name: Install pyMySQL
      pip:
        name: pymysql
        state: present
      ignore_errors: True

    - name: Create document root
      file:
        path: "/var/www/{{ http_host }}"
        state: directory
        owner: "{{ app_user }}"
        mode: '0755'
      
    - name: Set up Apache virtualhost
      template:
        src: "files/apache.conf.j2"
        dest: "/etc/apache2/sites-available/{{ http_conf }}"
      notify: Reload Apache

    - name: Enable new site
      shell: /usr/sbin/a2ensite {{ http_conf }}
      notify: Reload Apache

    - name: Disable default Apache site
      shell: /usr/sbin/a2dissite 000-default.conf
      when: disable_default
      notify: Reload Apache

  # MySQL Configuration
    - name: Sets the root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock
      ignore_errors: True
    
    - name: Set root user password
      mysql_user:
         name: root
         password: "{{ mysql_root_password }}"
         check_implicit_admin: yes
         login_user: admin
         login_password: "{{ mysql_root_password }}"
         state: present
    - name: Removes all anonymous user accounts
      mysql_user:
        name: ''
        host_all: yes
        state: absent
        login_user: root
        login_password: "{{ mysql_root_password }}"

    - name: Removes the MySQL test database
      mysql_db:
        name: test
        state: absent
        login_user: root
        login_password: "{{ mysql_root_password }}"

  # UFW Configuration
    - name: "UFW - Allow HTTP on port {{ http_port }}"
      ufw:
        rule: allow
        port: "{{ http_port }}"
        proto: tcp

  # PHP Info Page
    - name: Sets Up PHP Info Page
      template:
        src: "files/info.php.j2"
        dest: "/var/www/{{ http_host }}/info.php"

  handlers:
    - name: Reload Apache
      service:
        name: apache2
        state: reloaded

    - name: Restart Apache
      service:
        name: apache2
        state: restarted
```


## Conclusion
In this guide, we used Ansible to automate the process of installing and configuring Apache on Ubuntu 18.04.


### Contributors
[![Yogendra Pratap Singh][yogendra_avatar]][yogendra_homepage]<br/>[Yogendra Pratap Singh][yogendra_homepage] 

  [yogendra_homepage]: https://github.com/PratapSingh13
  [yogendra_avatar]: https://img.cloudposse.com/75x75/https://github.com/PratapSingh13.png
