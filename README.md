# The Oval App
## Ansible Deploy Overview
The server deployment uses Ansible for the three application components:
* DB (Postgres)
* Application Server (uWSGI)
* Web Server (nginx)

The ansible playbook is written for RHEL7 based distributions (CentOS, SciLi, etc) and Ubuntu 16.04.  It _may_ work for Ubuntu 14.04 and RHEL6, but there are a few systemd dependent tasks.

## Roles/Profiles
The three application components are divided in Ansible roles:
* [db](roles/db) 
* [app](roles/app) 
* [web](roles/web) 

Both postgres and nginx are installed from the "upstream" apt/yum repos (instead of the base CentOS or Ubuntu repos).  This provides a semi-consistent across operating systems.

There is a fourth [base](roles/base) role that applies to _all_ hosts.  This base role includes stuff like ntp, certbot, additional ssh keys, etc.

## Networking
The playbook assumes each host will have a "public" and "private" network.  Both public and private network interfaces are firewalled. Only tcp ports 80/443 are exposed on the public interface of each webserver. (All servers have tcp port 443 open for `certbot` certificate renewal.)

All `WEB <-> APP <-> DB` traffic occurs over the private network using TLS v1.2.

By default, all application servers will connect to the **first** database server listed in the `[db]` ansible group.  The current deployment is designed to utilize a single postgres DB instance.  Multiple DB servers can be deployed, however, only the first DB server will be used by the application servers.


## Inventory File

The default `site.yml` playbook will apply each role to the similarly named ansible group.  E.g. any hosts in the `[db]` ansible group will receive the `db` role.  An example inventory file with 7 hosts:
 * 1 x CentOS 7 database (postgres) server
 * 2 x Ubuntu 16.04 application (uwsgi) servers
 * 2 x CentOS 7 web (nginx) servers
 * 2 x Ubuntu 16.04 web (nginx) servers
```
[db]
db1.example.com	ansible_user=root 

[app]
app1.example.com ansible_user=root ansible_python_interpreter=/usr/bin/python3
app2.example.com ansible_user=root ansible_python_interpreter=/usr/bin/python3

[web]
web1.example.com ansible_user=root 
web2.example.com ansible_user=root 
web3.example.com ansible_user=root ansible_python_interpreter=/usr/bin/python3
web4.example.com ansible_user=root ansible_python_interpreter=/usr/bin/python3
```

Any servers that use python3 by default (e.g. Ubuntu 16.04) **must** have the `ansible_python_interpreter` variable set to `/usr/bin/python3`.



## Variables
### Global Variables
All variables are defined in [group_vars/all/vars.yml](group_vars/all/vars.yml) whereas all sensitive variables are stored in an encrypted ansible vault in [group_vars/all/vault.yml](group_vars/all/vault.yml).  For readability / searchability, sensitive variables (stored in `vars.yml` reference the corresponding `vault_` prefixed variable (stored in `vault.yml`) following [ansible best practices](http://docs.ansible.com/ansible/playbooks_best_practices.html#variables-and-vaults).  

Some application components (like the postgres version) can be overridden by these variables.  By default postgres 9.6 is installed/configured. However, overriding the `postgres_version` variable will change this behavior.

### Operating System Specific Variables
CentOS/Ubuntu specific variables are stored in [group_vars/os_RedHat.yml](group_vars/os_RedHat.yml) and [group_vars/os_Debian](group_vars/os_Debian) respectively.

Some packages (e.g. `python-devel` vs. `python-dev`) are named differently between distributions. These OS specific config files normalize variable names.


## Usage
Assuming a proper ansible inventory file is in place, use the `ansible-playbook` command to run the `site.yml` playbook using the `--ask-vault-pass` switch.


Example usage
```
[user@ansible-client oval-ansible]$ ansible-playbook site.yml --ask-vault-pass
Vault password: 

PLAY [import OS variables] *****************************************************

TASK [setup] *******************************************************************
ok: [db1.example.com]
ok: [app1.example.com]
ok: [app2.example.com]
ok: [web1.example.com]
ok: [web2.example.com]
ok: [web3.example.com]
ok: [web4.example.com]


TASK [include_vars] ************************************************************
ok: [db1.example.com]
ok: [app1.example.com]
ok: [app2.example.com]
ok: [web1.example.com]
ok: [web2.example.com]
ok: [web3.example.com]
ok: [web4.example.com]

PLAY [Deploy base] *************************************************************

TASK [setup] *******************************************************************
ok: [db1.example.com]
ok: [app1.example.com]
ok: [app2.example.com]
ok: [web1.example.com]
ok: [web2.example.com]
.
.
.
```
## Limitations
* The ansible scripts do not populate the DB with any data (e.g. admins, races, riders, etc), WIP
* A manual reboot might be required if SELinux is flipped from `disabled` to `enforcing` (this works if going from `permissive` to `enforcing`, however flipping from `disabled` -> `enforcing` requires a reboot.)
