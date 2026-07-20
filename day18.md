Cập nhật defaults/main.yml — thêm biến auth
```
mongo_version: "7.0"
mongo_data_dir: /data/db
mongo_port: 27017
mongo_bind_ip: "0.0.0.0"

mongo_admin_user: "admin"
mongo_admin_password: "{{ vault_mongo_admin_password }}"
mongo_app_db: "devops_practice"
mongo_app_user: "app_user"
mongo_app_password: "{{ vault_mongo_app_password }}"

# IP/CIDR được phép truy cập Mongo qua firewall - đổi đúng dải mạng cluster K8s của anh
allowed_k8s_cidr: "10.24.28.0/24"
```
- Tạo file chứa password, mã hóa bằng Ansible Vault
```
cd ~/devops-lab/day17-mongo
ansible-vault create group_vars/mongo_servers/vault.yml
```
```
vault_mongo_admin_password: "admin123!"
vault_mongo_app_password: "user123!"
```
- edit nếu cần 

```
ansible-vault edit group_vars/mongo_servers/vault.yml
```
- Thêm task tạo user vào tasks/main.yml
```
---
- name: Import MongoDB GPG key
  ansible.builtin.apt_key:
    url: https://pgp.mongodb.com/server-6.0.asc
    state: present

- name: Add MongoDB APT repository
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://repo.mongodb.org/apt/ubuntu {{ ansible_distribution_release }}/mongodb-org/{{ mongo_version }} multiverse"
    state: present
    filename: mongodb-org-6.0

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true

- name: Install MongoDB package
  block:
    - name: Install MongoDB package
      ansible.builtin.apt:
        name: mongodb-org
        state: present
  rescue:
    - name: Remove conflicting/broken MongoDB packages
      ansible.builtin.apt:
        name: "mongodb-org*"
        state: absent
        purge: true
        autoremove: true

    - name: Retry installing MongoDB package after cleanup
      ansible.builtin.apt:
        name: mongodb-org
        state: present

- name: Ensure pymongo is installed (required by community.mongodb.mongodb_user)
  ansible.builtin.apt:
    name: python3-pymongo
    state: present

- name: Ensure data directory exists with correct ownership
  ansible.builtin.file:
    path: "{{ mongo_data_dir }}"
    state: directory
    owner: mongodb
    group: mongodb
    mode: "0755"

- name: Deploy mongod.conf from template
  ansible.builtin.template:
    src: mongod.conf.j2
    dest: /etc/mongod.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart mongod

- name: Ensure ufw package is installed
  ansible.builtin.apt:
    name: ufw
    state: present

- name: Allow SSH so enabling ufw does not lock us out
  community.general.ufw:
    rule: allow
    port: "22"
    proto: tcp

- name: Allow MongoDB port only from K8s cluster CIDR (ufw)
  community.general.ufw:
    rule: allow
    port: "{{ mongo_port }}"
    proto: tcp
    src: "{{ allowed_k8s_cidr }}"

- name: Deny MongoDB port from all other sources (ufw)
  community.general.ufw:
    rule: deny
    port: "{{ mongo_port }}"
    proto: tcp

- name: "Enable ufw (default incoming policy left as-is: allow)"
  community.general.ufw:
    state: enabled
    policy: allow

- name: Enable and start mongod service
  ansible.builtin.systemd:
    name: mongod
    enabled: true
    state: started
- name: Wait for mongod to be ready before creating users
  ansible.builtin.wait_for:
    port: "{{ mongo_port }}"
    host: "127.0.0.1"
    delay: 3
    timeout: 30

- name: Probe whether MongoDB already enforces authentication
  ansible.builtin.command: >
    mongosh --quiet --host 127.0.0.1 --port {{ mongo_port }} --eval "db.adminCommand('ping')"
  register: mongo_auth_probe
  changed_when: false
  failed_when: false

- name: Set fact for whether auth is already enforced
  ansible.builtin.set_fact:
    mongo_auth_already_enabled: "{{ mongo_auth_probe.rc != 0 }}"

- name: Create admin user (before auth is enforced)
  community.mongodb.mongodb_user:
    login_host: localhost
    login_port: "{{ mongo_port }}"
    database: admin
    name: "{{ mongo_admin_user }}"
    password: "{{ mongo_admin_password }}"
    roles:
      - db: admin
        role: root
    state: present
  when: not mongo_auth_already_enabled

- name: Enable authentication in mongod.conf
  ansible.builtin.lineinfile:
    path: /etc/mongod.conf
    regexp: '^#?security:'
    line: "security:"
    state: present
  notify: Restart mongod

- name: Add authorization enabled under security
  ansible.builtin.lineinfile:
    path: /etc/mongod.conf
    insertafter: '^security:'
    line: "  authorization: enabled"
    state: present
  notify: Restart mongod

- name: Flush handlers to apply auth config now (needed before creating app user)
  ansible.builtin.meta: flush_handlers

- name: Create app-specific user with limited privileges
  community.mongodb.mongodb_user:
    login_host: localhost
    login_port: "{{ mongo_port }}"
    login_user: "{{ mongo_admin_user }}"
    login_password: "{{ mongo_admin_password }}"
    login_database: admin
    database: "{{ mongo_app_db }}"
    name: "{{ mongo_app_user }}"
    password: "{{ mongo_app_password }}"
    roles:
      - db: "{{ mongo_app_db }}"
        role: readWrite
    state: present
```
- cài collection này trước:
```
ansible-galaxy collection install community.mongodb
```
- chạy ansible 
```
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass
```

