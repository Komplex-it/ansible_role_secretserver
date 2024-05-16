Role Name
=========

A work-in-progress role to fetch passwords from Delinea Secret Server.

Requirements
------------

1. Lookup plugin: community.general.tss
   
    This role is dependent on the community.general.tss lookup plugin, which is part of the community.general collection.

    You might already have this collection installed if you are using the ansible package.It is not included in ansible-core.

    To check whether it is installed, run ansible-galaxy collection list.

    To install it, run: `ansible-galaxy collection install community.general`

2. Python dependency: python-tss-sdk

    The Delinea Secret Server Python SDK "python-tss-sdk" is required, see: https://pypi.org/project/python-tss-sdk/
    (community.general.tss requires it)

    To install it, run: `python -m pip install python-tss-sdk`

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
    ansible_controller: localhost # Use localhost or replace with actual ansible controller

  tasks:

    - name: Authenticate with SecretServer
      ansible.builtin.include_role:
        name: secretserver
        apply:
          delegate_to: "{{ ansible_controller }}"
          run_once: true
      vars:
        ss_hostname: <my.secretserver.eu> # Your Delinea Secret Server hostname
        ss_serviceaccount:
            name: <ServiceAccount> # username of an account with sufficient permissions to access secrets
            password: !vault |
              $ANSIBLE_VAULT;1.1;AES256
              <encrypted password> # ansible-vault encrypt_string --vault-password-file /path/to/file

    - name: dmz.test.local servers
      block:

        - name: Set name of secret to fetch
          ansible.builtin.set_fact:
            secret_name: administrator_{{ inventory_hostname }} # the name of the secret to fetch from secret server

        - name: Set path to the secrets
          ansible.builtin.set_fact:
            ss_secret_path: '\Server Admins\' # the actual folder path as seen on the secret server web interface

      when: '"dmz.test.local" in inventory_hostname'

    - name: test.local servers
      block:

        - name: Set name of secret to fetch
          ansible.builtin.set_fact:
            secret_name: PAM-DomainAdmin03

        - name: Set path to the secrets
          ansible.builtin.set_fact:
            ss_secret_path: '\Domain Admin\'

      when: 'inventory_hostname_short + ".test.local" in inventory_hostname'

    - name: .local servers
      block:

        - name: Set name of secret to fetch
          ansible.builtin.set_fact:
            secret_name: PAM-DomainAdmin04

        - name: Set path to the secrets
          ansible.builtin.set_fact:
            ss_secret_path: '\Domain Admin\'

      when: 'inventory_hostname_short + ".local" in inventory_hostname'


    - name: Fetch secrets
      ansible.builtin.include_role:
        name: secretserver
        tasks_from: fetch_secret
        apply:
          delegate_to: "{{ ansible_controller }}"

    - name: Set username and password for each server
      ansible.builtin.set_fact:
        ansible_user: "{{ username }}"
        ansible_password: "{{ secret }}"

    - name: Debug
      ansible.builtin.debug:
        msg:
          - "SecretServer Secret: {{ secret_name }}"
          - "Username: {{ ansible_user }}"
          - "Password: {{ ansible_password }}"
          - "Secret ID: {{ secret_id }}"

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

```


License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
