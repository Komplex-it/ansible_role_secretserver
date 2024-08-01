Ansible Role: secretserver
=========

A work-in-progress role to fetch passwords from Delinea Secret Server.

Requirements
------------

1. Lookup plugins: 

    community.general.tss  
    community.generel.json_query   

    This role is dependent on the community.general.tss lookup plugin and the community.generel.json_query plugin, which is part of the community.general collection.

    You might already have this collection installed if you are using the ansible package.It is not included in ansible-core.

    To check whether it is installed, run ansible-galaxy collection list.

    To install it, run: `ansible-galaxy collection install community.general`

2. Python dependencies: 

    python-tss-sdk  
    jmespath

    The Delinea Secret Server Python SDK "python-tss-sdk" is required, see: https://pypi.org/project/python-tss-sdk/ (community.general.tss requires it)
    "jmespath" is required for json queries.

    To install them, run: 
    `python -m pip install python-tss-sdk jmespath`

    In case the system python is different from what ansible uses, they can be installed like this:
    ```bash
    ansible --version | grep "python version" # the python that ansible uses.
    yum install python3.12-pip # install pip for that same python version, in this example python 3.12.
    python3.12 -m pip install python-tss-sdk jmespath # install the python modules under that same python version.
    ```

(3.) Kerberos requirements:

    ```bash
    yum install gcc python3.11-devel krb5-workstation krb5-devel krb5-libs
    python3.11 -m pip install pywinrm pywinrm[kerberos]
    *edit /etc/krb5.conf* # see: https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html#kerberos
    ```


Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

```yaml
---
- name: Using the secret server role
  hosts: all
  become: true
  gather_facts: false
  vars:
    ansible_controller: localhost

  vars_prompt:
    - name: serviceaccount_password
      prompt: Password for ServiceAccount
      private: true
      unsafe: true
    - name: serviceaccount2_password
      prompt: Password for ServiceAccount2
      private: true
      unsafe: true

  tasks:

    - name: Authenticate with SecretServer
      ansible.builtin.include_role:
        name: secretserver
        apply:
          delegate_to: "{{ ansible_controller }}"
          run_once: true
          no_log: true
      vars:
        ss_hostname: <secretserver hostname> # e.g. test.secretservercloud.eu
        ss_serviceaccounts:
         ServiceAccount:
           password: "{{ serviceaccount_password }}"
         ServiceAccount2:
           password: "{{ serviceaccount2_password }}"

## can be used instead of prompting for service account passwords.
## however, it is probably safer and smarter to just input them manually - they are accounts with access to secrets for all systems after all, and should be rotated.
#        ss_serviceaccounts:
#          ServiceAccount:
#           password: !vault |
#             $ANSIBLE_VAULT;1.1;AES256
#         ServiceAccount2:
#           password: !vault |
#             $ANSIBLE_VAULT;1.1;AES256


################################## Required vars - can be set elsewhere


## can be moved to e.g. group_vars/all/secretserver_accounts
#    - name: All secrets to fetch
#      ansible.builtin.set_fact:
#        ss_secrets:
#          - secret: PAM-DomainAdmin03
#            ss_secret_path: '\KITDemo\Domain Admin\'
#            ss_serviceaccount: ServiceAccount
#          - secret: PAM-DomainAdmin04
#            ss_secret_path: '\KITDemo\Domain Admin\'
#            ss_serviceaccount: ServiceAccount2
#        ss_secrets_unique:
#          - secret: administrator_{{ inventory_hostname }}
#            ss_secret_path: '\KITDemo\Server Admins\'
#            ss_serviceaccount: ServiceAccount
#      delegate_to: "{{ ansible_controller }}"

## can be moved under each group e.g. group_vars/dmz/secretserver_secrets or groups_vars/ss_secret_administrator/vars
#    - name: dmz.test.local servers
#      block:
#
#        - name: set secretserver vars for group
#          ansible.builtin.set_fact:
#            ss_secret: administrator_{{ inventory_hostname }}
#            unique_secret: true
#
#      when: '"dmz.test.local" in inventory_hostname'
#
#    - name: test.local servers
#      block:
#
#        - name: set secretserver vars for group
#          ansible.builtin.set_fact:
#            ss_secret: PAM-DomainAdmin03
#
#      when: 'inventory_hostname_short + ".test.local" in inventory_hostname'
#
#    - name: .local servers
#      block:
#
#        - name: set secretserver vars for group
#          ansible.builtin.set_fact:
#            ss_secret: PAM-DomainAdmin04
#
#      when: 'inventory_hostname_short + ".local" in inventory_hostname'

##################################


    - name: Combine all unique secrets
      set_fact:
        all_unique_secrets: "{{ all_unique_secrets | default([]) + hostvars[item]['ss_secrets_unique'] }}"
      delegate_to: "{{ ansible_controller }}"
      run_once: true
      with_items: "{{ groups['all'] }}"
      when: hostvars[item]['unique_secret'] is defined

    - name: merge lists
      set_fact:
        ss_secrets_all: "{{ ss_secrets + all_unique_secrets }}"
      delegate_to: "{{ ansible_controller }}"
      run_once: true
      when: all_unique_secrets is defined

    - name: Fetch secrets
      ansible.builtin.include_role:
        name: secretserver
        tasks_from: fetch_secret
        apply:
          delegate_to: "{{ ansible_controller }}"
          no_log: true
      loop: "{{ ss_secrets_all }}"
      loop_control:
        loop_var: outer_item
      run_once: true

    - name: Set username and password for each server
      ansible.builtin.set_fact:
        ansible_user: "{{ ss_secrets_all_final | json_query(query_username) }}"
        ansible_password: "{{ ss_secrets_all_final | json_query(query_pw) }}"
      vars:
        query_username: "[?secret == '{{ ss_secret }}'].username | [0]"
        query_pw: "[?secret == '{{ ss_secret }}'].pw | [0]"
      delegate_to: "{{ ansible_controller }}"


################################## Server Operations

    - name: Gather facts
      ansible.builtin.setup:

    - name: Install package
      ansible.builtin.package:
        name: httpd
        state: latest

#################################

    - name: Check-in
      ansible.builtin.include_role:
        name: secretserver
        tasks_from: check-in
        apply:
          delegate_to: "{{ ansible_controller }}"
          no_log: true
      loop: "{{ ss_secrets_all_final }}"
      loop_control:
        loop_var: outer_item
      run_once: true

```


License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
