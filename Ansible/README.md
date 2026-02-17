# RabbitMQ Cluster Deployment with Ansible (3 Nodes) -- Complete Guide

------------------------------------------------------------------------

# 🏗 Architecture Diagram

                        +------------------------+
                        |   Ansible Control VM   |
                        |------------------------|
                        |  Runs Playbooks        |
                        |  SSH to all nodes      |
                        +-----------+------------+
                                    |
            -------------------------------------------------------
            |                     |                     |
    +---------------+   +---------------+   +---------------+
    |   rabbit1     |   |   rabbit2     |   |   rabbit3     |
    |---------------|   |---------------|   |---------------|
    | Master Node   |   | Cluster Node  |   | Cluster Node  |
    | 5672  AMQP    |   | 5672  AMQP    |   | 5672  AMQP    |
    | 15672 UI      |   | 15672 UI      |   | 15672 UI      |
    | 25672 Cluster |   | 25672 Cluster |   | 25672 Cluster |
    +---------------+   +---------------+   +---------------+

------------------------------------------------------------------------

# 📁 Project Structure

    rabbitmq-project/
    ├── inventory.ini
    ├── site.yml
    └── roles/
        └── rabbitmq/
            ├── defaults/main.yml
            ├── handlers/main.yml
            ├── tasks/
            │   ├── main.yml
            │   ├── install.yml
            │   ├── config.yml
            │   ├── cluster.yml
            │   ├── users.yml
            │   └── verify.yml
            ├── templates/rabbitmq.conf.j2
            └── vars/

------------------------------------------------------------------------

# 📦 inventory.ini

    [rabbitmq]
    rabbit1 ansible_host=10.0.1.10
    rabbit2 ansible_host=10.0.1.11
    rabbit3 ansible_host=10.0.1.12

    [rabbitmq:vars]
    ansible_user=ec2-user
    ansible_ssh_private_key_file=~/.ssh/id_rsa

------------------------------------------------------------------------

# ▶ site.yml

    ---
    - name: Install and Configure RabbitMQ Cluster
      hosts: rabbitmq
      become: true
      roles:
        - rabbitmq

------------------------------------------------------------------------

# 🧠 defaults/main.yml

    ---
    rabbitmq_master: "rabbit1"
    rabbitmq_admin_user: "myadmin"
    rabbitmq_admin_pass: "StrongPass123!"
    rabbitmq_delete_guest: true
    rabbitmq_enable_management: true

------------------------------------------------------------------------

# 🔧 handlers/main.yml

    ---
    - name: restart rabbitmq
      systemd:
        name: rabbitmq-server
        state: restarted

------------------------------------------------------------------------

# 🛠 tasks/install.yml

    ---
    - name: Install Erlang and RabbitMQ
      yum:
        name:
          - erlang
          - rabbitmq-server
        state: present

    - name: Enable and start RabbitMQ
      systemd:
        name: rabbitmq-server
        enabled: yes
        state: started

------------------------------------------------------------------------

# ⚙ tasks/config.yml

    ---
    - name: Deploy rabbitmq.conf
      template:
        src: rabbitmq.conf.j2
        dest: /etc/rabbitmq/rabbitmq.conf
      notify: restart rabbitmq

    - name: Enable management plugin
      command: rabbitmq-plugins enable rabbitmq_management
      when: rabbitmq_enable_management
      changed_when: false
      notify: restart rabbitmq

------------------------------------------------------------------------

# 🔗 tasks/cluster.yml

    ---
    - name: Stop RabbitMQ app (non-master)
      command: rabbitmqctl stop_app
      when: inventory_hostname != rabbitmq_master

    - name: Reset node (non-master)
      command: rabbitmqctl reset
      when: inventory_hostname != rabbitmq_master

    - name: Join cluster
      command: rabbitmqctl join_cluster rabbit@{{ rabbitmq_master }}
      when: inventory_hostname != rabbitmq_master

    - name: Start RabbitMQ app
      command: rabbitmqctl start_app
      when: inventory_hostname != rabbitmq_master

------------------------------------------------------------------------

# 👤 tasks/users.yml

    ---
    - name: Add admin user (master only)
      command: rabbitmqctl add_user {{ rabbitmq_admin_user }} {{ rabbitmq_admin_pass }}
      when: inventory_hostname == rabbitmq_master
      ignore_errors: yes

    - name: Set admin tag
      command: rabbitmqctl set_user_tags {{ rabbitmq_admin_user }} administrator
      when: inventory_hostname == rabbitmq_master

    - name: Set permissions
      command: rabbitmqctl set_permissions -p / {{ rabbitmq_admin_user }} ".*" ".*" ".*"
      when: inventory_hostname == rabbitmq_master

    - name: Delete guest user
      command: rabbitmqctl delete_user guest
      when: inventory_hostname == rabbitmq_master and rabbitmq_delete_guest
      ignore_errors: yes

------------------------------------------------------------------------

# 🔎 tasks/verify.yml

    ---
    - name: Check cluster status
      command: rabbitmqctl cluster_status
      when: inventory_hostname == rabbitmq_master
      register: cluster_status

    - name: Show cluster status
      debug:
        var: cluster_status.stdout
      when: inventory_hostname == rabbitmq_master

------------------------------------------------------------------------

# 🔐 Required AWS Security Group Ports

  Port          Purpose
  ------------- ---------------------------
  4369          Erlang Port Mapper
  25672         Inter-node clustering
  5672          AMQP
  15672         Management UI
  35672-35682   Erlang distribution range

------------------------------------------------------------------------

# ✅ Final Verification

    sudo rabbitmqctl cluster_status

Expected:

    {disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}

------------------------------------------------------------------------

# 🚀 Result

✔ 3-node RabbitMQ cluster\
✔ Management UI enabled\
✔ Clean Ansible role structure\
✔ Production-ready deployment

------------------------------------------------------------------------

End of Document.


---

# 📊 Deployment Screenshots

## 🐰 RabbitMQ Management UI (Cluster Running)

![RabbitMQ Management UI](rabbitmq_management_ui.png)

This screenshot shows:
- RabbitMQ 4.2.3
- Erlang 27.x
- Cluster name: rabbit@rabbit1
- Node statistics active
- Management UI accessible via port 15672

---

## ☁ AWS EC2 Instances (All Running)

![AWS EC2 Instances](aws_ec2_instances.png)

This screenshot confirms:
- 3 RabbitMQ nodes (rabbit1, rabbit2, rabbit3)
- 1 Ansible control server
- All instances running
- Status checks passed (3/3)

---
