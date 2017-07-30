---
layout: post 
author: shalou
title:  "ansible部署mysql" 
category: ansible
tag: ["ansible"]
---
install.yaml

```
- hosts: all
  gather_facts: false
  user: root
  roles:
  - { role: mariadb, install: true, uninstall: false }
```

<!-- more -->

uninstall.yaml

```
- hosts: all
  gather_facts: false
  user: root
  roles:
  - { role: mariadb, install: false, uninstall: true }
```

roles/mariadb/tasks/main.yaml

```
- name: Get stats mysql data dir
  stat:
    path: /var/lib/mysql
  register: mysql

- name: check mysql
  set_fact:
    exist: true
  when: mysql.stat.isdir is defined and mysql.stat.isdir

- name: Install Mariadb
  yum: name={{ item }} state=present
  with_items:
  - mariadb-server
  - MySQL-python
  when: install==true and exist is undefined

- name: enable and start mariadb
  service:
    name: mariadb
    enabled: true
    state: started
  when: install==true and exist is undefined

- name: Update MySQL root password for all root accounts
  mysql_user: name=root host={{ item }} password={{ mysql_root_pass }} state=present
  with_items:
  - "%"
  - 127.0.0.1
  - ::1
  - localhost
  when: install==true and exist is undefined

- name: Copy the root credentials as .my.cnf file
  template:
    src: my.cnf.j2
    dest: ~/.my.cnf
  when: install==true

- name: Ensure Anonymous user is not in the database
  mysql_user:
    name: ''
    host_all: yes
    state: absent
  when: install==true and exist is undefined

- name: Remove the test database
  mysql_db:
    name: test
    state: absent
  when: install==true and exist is undefined

- name: Stop and Disable Mariadb
  service:
    name: mariadb
    state: stopped
    enabled: false
  ignore_errors: yes
  when: uninstall==true

- name: Uninstall Mariadb
  yum: name={{ item }} state=absent
  with_items:
  - mariadb-server
  - MySQL-python
  when: uninstall==true

- name: Remove Mariadb data
  file:
    path: /var/lib/mysql
    state: absent
  when: uninstall==true

- name: Remove credential
  file:
    path: ~/.my.cnf
    state: absent
  when: uninstall==true
```

roles/mariadb/vars/main.yaml

```
mysql_root_pass: xxxxxxx
```

roles/mariadb/templates/my.cnf.j2


```
[client]
user=root
password={{ mysql_root_pass }}
```
