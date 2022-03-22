This is an ansible script for deploying the mediawiki on RHEL.


Launch 1 VM with t2 medium tier VM.
Login into VM as Root user
sudo -i

update the repository
sudo yum update

Create the user called ansible
adduser ansible

change the password
passwd ansible

run the below command with password authentication
vi /etc/ssh/sshd_config
passwordauthentication yes

restart the sshd service
service sshd restart


run the command visudo and add the user with sudo privileges.
ansible ALL= (ALL:ALL) ALL

exit of the sudo user

login with ansible user
su ansible

give the password and then login successfully

run the below command for update
sudo yum update

again give the password for authentication.

Now we are going to change the password authentication.
ansible ALL= (ALL:ALL)  NOPASSWD: ALL

Once again login into ansible user and this time will not ask for the password.
sudo yum update

now exit out of all the user and try to login with ssh with below command
ssh ansible@publicip

now Install the ansible on Ansible control server
sudo yum install ansible

verify the ansible version
ansible --version

Now login into the invevntory file of ansible. This is the default inventory file of ansible. we can change this inventory file or create new one.
vi /etc/ansible/hosts

we are going to copy this file and create new inventory file.
sudo cp hosts hosts.orig

This new hosts file is empty and write localhost and save
run the below command.
ansible -m ping all

This ansible server is not reaching because of the authentication issue. so now we are going to create the ssh key for the authentication.
sss-keygen
cd /home/ansible
cd .ssh/

Here we can verify both the public and private ssh key is present.

now this key is copy to the localhost.
ssh-copy-id ansible@localhost

now we can verify ansible server reaching out to the localhost.
ansible -m ping all.

Create one new file demo.yaml and copy the below content for install the httpd, mysql, and mediawiki.
---
- hosts: all
  become: yes
  tasks:
    - name: install httpd
      dnf:
        name: httpd
        state: latest
        update_cache: yes
    - name: start the service
      service:
        name: httpd
        enabled: yes
    - name: download epel release
      get_url:
        url:  https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
        dest: /home/ansible/
    - name: install repository
      get_url:
        url:  https://rpms.remirepo.net/enterprise/remi-release-7.rpm
        dest: /home/ansible/
    - name: Enable remi repo
      dnf:
        enablerepo: remi-7.4
    - name: Install PHP core"
      dnf:
        name: "{{ item }}"
        state: latest
      loop:
      - php
      - php-mysqlnd
      - php-gd
      - php-xml
      - php-mbstring
      - php-json

    - name: restart httpd
      service:
        name: httpd
        state: restarted
    - name: Update DNF Package repository cache
      dnf:
         update_cache: True
    - name: Install MySQL server on RHEL
      dnf:
        name: mysql-server
        state: present
    - name: Install MySQL client on RHEL
      dnf:
        name: mysql
        state: present
    - name: Make sure mysqld service is running
      service:
        name: mysqld
        state: started
        enabled: True

    - name: Install python3-PyMySQL library
      dnf:
        name: python3-PyMySQL
        state: present
    - name: Set MySQL root Password
      mysql_user:
        name: "root"
        host: "localhost"
        login_user: "root"
        login_password: "password"
        password: "password"
        check_implicit_admin: yes
        priv: "*.*:ALL,GRANT"
        state: present
        update_password: always
    - name: create new wiki user
      mysql_user:
        name: "wikiuser"
        password: "password33"
        login_user: "root"
        login_password: "password"
        priv: "*.*:ALL,GRANT"
        host: "localhost"
        state: present
    - name: Create database
      mysql_db:
        name: "wikidatabase"
        login_user: wikiuser
        login_password: "password33"
        state: present
    - name: collect all information
      mysql_info:
        login_user: "wikiuser"
        login_password: "password33"
        filter:
        - databases
        - version
    - name: Make sure mysqld service is running
      service:
        name: mysqld
        state: started
        enabled: True
    - name: Download mediawiki tarball
      get_url:
        url: https://releases.wikimedia.org/mediawiki/1.37/mediawiki-1.37.1.tar.gz
        dest: /home/ansible/
    - name: Download signature file
      get_url:
         url: https://releases.wikimedia.org/mediawiki/1.37/mediawiki-1.37.1.tar.gz.sig
         dest: /home/ansible/
    - name: Extract mediawiki
      ansible.builtin.unarchive:
        src: /home/ansible/mediawiki-1.37.1.tar.gz
        dest: /var/www/
        remote_src: yes
    - name: change tar file
      command: ln -s mediawiki-1.37.1/ mediawiki
    - name: Change the location of tar file  
      command: chown -R apache:apache /var/www/mediawiki

    - name: restart apache
      service:
        name: httpd
        state: restarted
    - name: Enforcing SELinux
      selinux:
        state: enforcing
        policy: targeted
    - name: Apply new SELinux file context to filesystem
      shell: restorecon -FR /var/www/mediawiki-1.37.1/
    - name: Apply new SELinux file context to filesystem
      shell: ln -s mediawiki-1.37.1/ mediawiki
    - name: mediawiki deploy
      command: chown -R apache:apache /var/www/html/mediawiki

After copied the content save and exit.
run the below command for execute the script

ansible-playbook demo.yaml

and then access the mediawiki on http://publicip/mediawiki

This mediawiki will not fully loaded may be there have some extention issue.
