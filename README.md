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

3. Kerberos requirements (only required for kerberos logins):

    ```bash
    yum install gcc python3.11-devel krb5-workstation krb5-devel krb5-libs
    python3.11 -m pip install pywinrm pywinrm[kerberos]
    *edit /etc/krb5.conf* # see: https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html#kerberos
    ```


Role Variables
--------------

1. Define list with secrets to fetch including path and service account in group_vars/all/xx (will probably be moved to roles/secretserver/vars/main.yml at some point, but that doesnt work for now):

    Example of a group_vars/all/secretserver.yml:

    ```yaml
    ---
    ss_secrets:
      - secret: PAM-DomainAdmin01
        ss_secret_path: '\demo.local\Domain Admins\'
        ss_serviceaccount: ServiceAccount
      - secret: PAM-DmzDomainAdmin01
        ss_secret_path: '\dmz.demo.local\Domain Admins\'
        ss_serviceaccount: ServiceAccount
    ss_secrets_unique:
      - secret: Administrator_{{ inventory_hostname }}
        ss_secret_path: '\workgroup.local\Server Admins\'
        ss_serviceaccount: ServiceAccount
    ```

2. Define what secret each server/group should use, either directly in the inventory or via group vars.

    Example of setting it in an inventory ini file:

    ```ini
    [demo_local]
    app01.demo.local
    db01.demo.local

    [demo_local:vars]
    ss_secret=PAM-DomainAdmin01


    [dmz_demo_local]
    web01.dmz.demo.local

    [dmz_demo_local:vars]
    ss_secret=PAM-DmzDomainAdmin01


    [workgroup_local]
    bck01.workgroup.local

    [workgroup_local:vars]
    ss_secret=Administrator_{{ inventory_hostname }}
    ```

3. Optional settings in ansible.cfg.

    Some recommended settings in ansible.cfg:

    ```cfg
    [defaults]
    display_skipped_hosts = false # there will be a lot of output from skipped tasks without this.
    ```


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

```yaml
---
- name: Delinea Secret Server demo
  hosts: all
  gather_facts: false
  force_handlers: true
  vars:
    ansible_controller: localhost

  vars_prompt:
    - name: serviceaccount_password
      prompt: Password for ServiceAccount
      private: true
      unsafe: true
#    - name: serviceaccount2_password
#      prompt: Password for ServiceAccount2
#      private: true
#      unsafe: true

  tasks:

    - name: Fetch credentials from Delinea Secret Server
      ansible.builtin.include_role:
        name: secretserver
      vars:
        ss_hostname: demokit.secretservercloud.eu
        ss_serviceaccounts:
          ServiceAccount:
            password: "{{ serviceaccount_password }}"
#          ServiceAccount2:
#            password: "{{ serviceaccount2_password }}"


    - name: Gather facts
      ansible.builtin.setup:

#    - name: Install package
#      ansible.builtin.package:
#        name: httpd
#        state: latest

```


License
-------

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
