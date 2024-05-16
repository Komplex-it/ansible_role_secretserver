Role Name
=========

A brief description of the role goes here.

Requirements
------------

1. Lookup plugin
   
  This role is dependent on the community.general.tss lookup plugin, which is part of the community.general collection.
  You might already have this collection installed if you are using the ansible package. It is not included in ansible-core. To check whether it is installed, run ansible-galaxy collection list.
  To install it, use: ansible-galaxy collection install community.general. You need further requirements to be able to use this lookup plugin, see Requirements for details.

2. Python dependency

  The Delinea Secret Server Python SDK "python-tss-sdk" is also required: https://pypi.org/project/python-tss-sdk/
  (community.general.tss requires it)

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

---
- name: Using the secret server role
  hosts: all
  become: true
  gather_facts: false

  tasks:

    - name: Authenticate with SecretServer
      ansible.builtin.include_role:
        name: secretserver
        apply:
          delegate_to: localhost
          run_once: true
      vars:
        ss_hostname: my.secretserver.eu
        ss_serviceaccount:
            name: <username of a secret server account with suffiecient permissions>
            password: !vault |
              $ANSIBLE_VAULT;1.1;AES256
              <here goes the encrypted password of the account>

    - name: dmz.test.local servers
      block:

        - name: Set name of secret to fetch
          ansible.builtin.set_fact:
            secret_name: administrator_{{ inventory_hostname }}

        - name: Set path to the secrets
          ansible.builtin.set_fact:
            ss_secret_path: '\Server Admins\'

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
          delegate_to: localhost

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
#          - "{{ temp_secret }}"

################################## Server Operations

    - name: Gather facts
      ansible.builtin.setup:

#################################

    - name: Check-in
      ansible.builtin.include_role:
        name: secretserver
        tasks_from: check-in
        apply:
          delegate_to: localhost



License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
