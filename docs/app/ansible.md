ansible
===

## ansible のインストール

[Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) によると、Python パッケージでインストールする方法が公式手順のようです。

- Python パッケージのインストール

    - Python のインストール

        ```bash
        sudo apt install -y python3 python3-venv
        ```

    - Python 仮想環境の準備

        ```bash
        python3 -m venv $HOME/venv
        source $HOME/venv/bin/activate
        ```

    - ansible のインストール

        ```bash
        pip install ansible
        ```

- Debian Linux

    ```bash
    sudo apt update
    sudo apt install -y ansible
    ```

## ansible 実行前準備

1. sudo コマンドのインストール (sudo がインストールされていない場合)

    ```bash
    su -
    apt update \
        && apt install -y sudo
    ```

2. 管理用アカウント作製

    ```bash
    sudo useradd -g 100 -G sudo \
        -d /home/admin -m \
        -s /bin/bash admin
    sudo passwd admin
    ```

3. admin ユーザの sudo 設定

    ```bash
    echo -n "admin ALL=(ALL) NOPASSWD:ALL" \
        | sudo tee /etc/sudoers.d/admin
    ```

## 実行例

### ローカルで実行

- hosts.ini

    ```ini
    [local]
    localhost ansible_connection=local ansible_become=true
    ```

- ansible の実行方法

    ```bash
    ansible-playbook -i hosts.ini config.yaml
    ```

#### Playbook の内容

- config.yaml

    ```yaml
    - import_playbook: install_packages.yaml
    - import_playbook: enable_services.yaml
    - import_playbook: locales.yaml
    - import_playbook: cleanup.yaml
    ```

- install_packages.yaml

    ```yaml
    - name: Install required packages on Debian
      hosts: localhost
      connection: local
      become: true
      tasks:
        - name: Update apt cache
          ansible.builtin.apt:
            update_cache: yes
    
        - name: Install base tools
          ansible.builtin.apt:
            name:
              - ssh
              - avahi-daemon
              - vim
              - sudo
              - git
              - exuberant-ctags
              - man-db
              - manpages-ja
              - locales
              - bash-completion
            state: present
    
        - name: Install archive tools
          ansible.builtin.apt:
            name:
              - zip
              - unzip
              - p7zip-full
            state: present
    
        - name: Install development tools
          ansible.builtin.apt:
            name:
              - build-essential
              - make
              - cmake
              - pkg-config
              - gdb
            state: present
    
        - name: Install additional useful tools
          ansible.builtin.apt:
            name:
              - curl
              - wget
              - net-tools
              - htop
              - tree
            state: present
    ```
    
- enable_services.yaml
    
    ```yaml
    - name: Enable and start required services on Debian
      hosts: localhost
      connection: local
      tasks:
        - name: Ensure sshd is enabled and running
          ansible.builtin.service:
            name: ssh
            state: started
            enabled: yes
    
        - name: Ensure avahi-daemon is enabled and running
          ansible.builtin.service:
            name: avahi-daemon
            state: started
            enabled: yes
    ```
    
    
- locales.yaml
    
    ```yaml
    - name: Set the locale to ja_JP.UTF-8
      hosts: localhost
      connection: local
      tasks:
        - name: Generate ja_JP.UTF-8 locale
          ansible.builtin.command: locale-gen ja_JP.UTF-8
          args:
            creates: /usr/lib/locale/ja_JP.utf8
    
        - name: Update default locale
          ansible.builtin.lineinfile:
            path: /etc/default/locale
            regexp: '^LANG='
            line: 'LANG=ja_JP.UTF-8'
    ```

- cleanup.yaml

    ```yaml
    - name: Remove file with Ansible
      hosts: localhost
      connection: local
      tasks:
        - name: Delete /etc/sudoers.d/admin
          ansible.builtin.file:
            path: /etc/sudoers.d/admin
            state: absent
    
    #- name: Remove admin account
    #  become: true
    #  user:
    #    name: admin
    #    state: absent
    #    remove: yes   # ホームディレクトリやメールスプールも削除
    ```


### リモートで実行

- hosts.ini

    ```ini
    [local]
    127.0.0.1 ansible_user=admin ansible_connection=ssh
    ```

- ansible の実行方法

    ```bash
    ansible-playbook -i hosts.ini config.yaml --ask-pass
    ```

#### Playbook の内容

- config.yaml

    ```yaml
    - import_playbook: install_packages.yaml
    - import_playbook: enable_services.yaml
    - import_playbook: locales.yaml
    - import_playbook: cleanup.yaml
    ```

- install_packages.yaml

    ```yaml
    - name: Install required packages on Debian
      hosts: all
      become: true
      tasks:
        - name: Update apt cache
          ansible.builtin.apt:
            update_cache: yes
    
        - name: Install base tools
          ansible.builtin.apt:
            name:
              - ssh
              - avahi-daemon
              - vim
              - sudo
              - git
              - exuberant-ctags
              - man-db
              - manpages-ja
              - locales
              - bash-completion
            state: present
    
        - name: Install archive tools
          ansible.builtin.apt:
            name:
              - zip
              - unzip
              - p7zip-full
            state: present
    
        - name: Install development tools
          ansible.builtin.apt:
            name:
              - build-essential
              - make
              - cmake
              - pkg-config
              - gdb
            state: present
    
        - name: Install additional useful tools
          ansible.builtin.apt:
            name:
              - curl
              - wget
              - net-tools
              - htop
              - tree
            state: present
    ```
    
- enable_services.yaml
    
    ```yaml
    - name: Enable and start required services on Debian
      hosts: all
      become: true
      tasks:
        - name: Ensure sshd is enabled and running
          ansible.builtin.service:
            name: ssh
            state: started
            enabled: yes
    
        - name: Ensure avahi-daemon is enabled and running
          ansible.builtin.service:
            name: avahi-daemon
            state: started
            enabled: yes
    ```
    
    
- locales.yaml
    
    ```yaml
    - name: Set the locale to ja_JP.UTF-8
      hosts: all
      become: true
      tasks:
        - name: Generate ja_JP.UTF-8 locale
          ansible.builtin.command: locale-gen ja_JP.UTF-8
          args:
            creates: /usr/lib/locale/ja_JP.utf8
    
        - name: Update default locale
          ansible.builtin.lineinfile:
            path: /etc/default/locale
            regexp: '^LANG='
            line: 'LANG=ja_JP.UTF-8'
    ```

- cleanup.yaml

    ```yaml
    - name: Remove admin sudo NOPASSWD rule
      become: true
      file:
        path: /etc/sudoers.d/admin
        state: absent
    
    #- name: Remove admin account
    #  become: true
    #  user:
    #    name: admin
    #    state: absent
    #    remove: yes   # ホームディレクトリやメールスプールも削除
    ```
