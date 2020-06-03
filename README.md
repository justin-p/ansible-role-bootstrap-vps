# Bootstrap-host

![Github Actions](https://img.shields.io/github/workflow/status/justin-p/ansible-bootstrap-host/CI?label=Github%20Actions&logo=github&style=flat-square)
![Travis](https://img.shields.io/travis/justin-p/ansible-bootstrap-host?label=Travis&logo=travis&style=flat-square)

A Ansible role I build for quickly configuring and hardening a new VPS.  

You probably don't want to use the default password/salt values. Overwrite these in each project with unique values and store them securely with Ansible Vault. There are also some other variables you want to overwrite. These are listed in the Example Playbook.

Huge thanks to developers of the roles listed below:

- dev-sec.ssh-hardening
- dev-sec.os-hardening
- jnv.unattended-upgrades
- oefenweb.fail2ban
- oefenweb.postfix
- oefenweb.haveged
- hwwilliams.logwatch


## Role Variables

| Variable                                 | Description                                                                                                                                                    | Default value                                                              |
| :----------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| bsh_host_fqdn                        | Hostname used for postifx and logwatch. This should be set to a FQDN.                                                                                          | host.domain.tld                                                            |
| bsh_sshkey_folder                    | Local folder where the ssh keypair for bsh_admin_public_key_path is stored.                                                                                    | ~/.ssh                                                                     |
| bsh_root_username                    | The initial username on the system. If your provider already sets up a non-root account use that information here and under any bsh_admin variables.           | root                                                                       |
| bsh_root_password                    | The password for the initial user. If your provider already sets up a non-root account use that information here and under any bsh_admin variables.            | 293f557d663c0da6f17b5659c2fd32f3!                                          |
| bsh_root_password_salt               | The password salt for the initial user. If your provider already sets up a non-root account use that information here and under any bsh_admin variables.       | aeb0753e9ea66ac8a0a64248cf8fd88d!                                          |
| bsh_admin_username                   | The username for the admin user. This is your non-root administration account                                                                                  | admin                                                                      |
| bsh_admin_password                   | The password for the admin user.                                                                                                                               | 3053da8d3858f9f0d3e4f52ebdc17fb3@                                          |
| bsh_admin_password_salt              | The password salt for the admin user.                                                                                                                          | 174ca66f4c677c7653037a722091db37%                                          |
| bsh_admin_public_key_path            | The path to the local public key that should be added to the admin user.                                                                                       | "{{ bsh_sshkey_folder }}/id_rsa.pub"                                       |
| bsh_admin_mailto                     | A e-mail address where you would like to receive e-mails that would normally be send to root account.                                                          | admin@domain.tld                                                           |
| bsh_postfix_hostname                 | The hostname used by postfix when deliverying e-mail. This should be set to a FQDN.                                                                            | "{{ bsh_host_fqdn }}"                                                      |
| bsh_postfix_aliases                  | Setup a alias to forward all the e-mails from root to bsh_admin_mailto.                                                                                        | - { user: root, alias: "{{ bsh_admin_mailto }}" }                          |
| bsh_packges_to_install               | Packages to install during bootstapping.                                                                                                                       | [vim, htop, ranger, tmux, ncdu]                                            |
| bsh_ufw_rules                        | Firewall rules that should be created.                                                                                                                         | - { rule: allow, interface: eth0, proto: tcp, port: '22', direction: in }  |
|                                      |                                                                                                                                                                | - { rule: allow, interface: eth0, proto: tcp, port: '80', direction: in }  |
|                                      |                                                                                                                                                                | - { rule: allow, interface: eth0, proto: tcp, port: '443', direction: in } |
| bsh_sysctl_overwrite                 | Overwrite values that are set duding sysctl hardening. For a full list of options see [dev-sec.os-hardening](https://github.com/dev-sec/ansible-os-hardening). | - { net.ipv4.ip_forward: 0 }                                               |
| bsh_unattended_automatic_reboot      | If the server should reboot after installing automatic updates.                                                                                                | true                                                                       |
| bsh_unattended_automatic_reboot_time | At what time the server should reboot after installing automatic updates.                                                                                      | "04:00"                                                                    |
| bsh_unattended_mail                  | The e-mail address used to send status e-mails to regarding the installation of automatic updates.                                                             | "{{ bsh_admin_mailto }}"                                                   |
| bsh_unattended_mail_only_on_error    | If the server should only send e-mails if it encounters errors when installing automatic updates.                                                              | true                                                                       |
| bsh_fail2ban_services                | The services Fail2Ban should monitor.                                                                                                                          | - { name: sshd, port: 22, maxretry: 3, bantime: -1 }                       |
| bsh_fail2ban_ignoreips               | The IPs added to the ignore list of Fail2Ban                                                                                                                   | [127.0.0.1/8]                                                              |
| bsh_fail2ban_bantime                 | The time to ban IPs for (-1 = permanent)                                                                                                                       | -1                                                                         |
| bsh_fail2ban_maxretry                | The amount of retries allowed before banning an ip.                                                                                                            | 5                                                                          |
| bsh_logwatch_mailfrom                | The e-mail address to mail from when sending logwatch e-mails                                                                                                  | Logwatch@{{ bsh_host_fqdn }}                                               |
| bsh_logwatch_mailto                  | The e-mail address that should receive the logwatch e-mails.                                                                                                   | root                                                                       |
| bsh_logwatch_detail                  | The level of detail the logwatch e-mails should use.                                                                                                           | high                                                                       |
| bsh_logwatch_range                   | The range that logwatch should use for the e-mails.                                                                                                            | today                                                                      |
| bsh_logwatch_output                  | The output logwatch should use for the e-mails.                                                                                                                | stdout                                                                     |
| bsh_logwatch_format                  | The format logwatch should use for the e-mails.                                                                                                                | html                                                                       |
| bsh_ssh_allow_users                  | The users that are allowed to login with ssh.                                                                                                                  | "{{ bsh_admin_username }}"                                                 |

## Example Playbook

```yaml
---
# We use this check to determine if we already hardened the server, if so we switch to the admin account.
# This ensures we can run this bootstrap role multiple times since root login will be disabled after the first run.
# Make sure you add `bsh_fail2ban_ignoreips` with your external IP since this can lock you out after configuring fail2ban.
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

## License

MIT

## Author Information

- Justin Perdok ([@justin-p](https://github.com/justin-p/))
