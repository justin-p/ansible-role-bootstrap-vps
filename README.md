Role Name
=========

A brief description of the role goes here.

Requirements
------------

TBD

Role Variables
--------------

TBD

Dependencies
------------

none

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yaml
---
# We use this check to determine if we already hardened the server, if so we switch to the admin account.
# This ensures we can run this bootstrap role multiple times since root login will be disabled after the first run.
# Make sure you add `ignoreips_fail2ban` with your external IP since this can lock you out after configuring fail2ban.
- hosts: all
  gather_facts: no
  vars_files:
    - ./vars/all.yml
    - ./vars/all_secrets.yml
  vars:
    users:
    - {username: "{{ root_username }}", sshkey: "{{ root_private_key_path }}", password: "{{ root_password }}" }
    - {username: "{{ admin_username }}", sshkey: "{{ admin_private_key_path }}", password: "{{ admin_password }}" }
  tasks:
  - name: Test if we can logon with the root account and the admin account
    shell: ssh -o User={{ item.username }} -o ConnectTimeout=10 -o StrictHostKeyChecking=no -i {{ item.sshkey }} {{ ansible_host }} /bin/true
    register: check_user
    connection: local
    ignore_errors: yes
    changed_when: False
    with_items: "{{ users }}"
    no_log: True

- hosts: all
  name: Bootstrap host
  vars_files:
    - ./vars/all.yml
    - ./vars/all_secrets.yml
  vars:
    ansible_user: "{{ (check_user.results | rejectattr('failed', 'equalto', true)|first).item.username }}"
    ansible_become_pass: "{{ (check_user.results | rejectattr('failed', 'equalto', true)|first).item.password }}"
    ansible_ssh_private_key_file: "{{ (check_user.results | rejectattr('failed', 'equalto', true)|first).item.sshkey }}"
  pre_tasks:
    - name: Wait for apt to be unlocked.
      become: true
      shell: while sudo fuser /var/lib/dpkg/{{ item }} >/dev/null 2>&1; do sleep 1; done;
      with_items:
        - lock
        - lock-frontend
      changed_when: False
    - name: Update apt cache if needed.
      apt:
        update_cache: yes
        cache_valid_time: 3600
      become: yes
  tasks:
      - include_role:
          name: bootstrap-host
        vars:
          bsh_host_fqdn: "{{ host_fqdn }}"
          bsh_root_username: "{{ root_username }}"
          bsh_root_password: "{{ root_password }}"
          bsh_root_password_salt: "{{ root_password_salt }}"
          bsh_admin_username: "{{ admin_username }}"
          bsh_admin_password: "{{ admin_password }}"
          bsh_admin_password_salt: "{{ admin_password_salt }}"
          bsh_docker_username: "{{ docker_username }}"
          bsh_docker_password: "{{ docker_password }}"
          bsh_docker_password_salt: "{{ docker_password_salt }}"
          bsh_admin_public_key_path: "{{ admin_public_key_path }}"
          bsh_admin_mailto: "{{ admin_mailto }}"
          bsh_fail2ban_ignoreips: "{{ ignoreips_fail2ban }}"
          bsh_sysctl_overwrite:
            - { net.ipv4.ip_forward: 1 }
          bsh_ssh_allow_users: "{{ admin_username }} {{ docker_username }}"
```

License
-------

MIT

Author Information
------------------

Justin Perdok
