CTF Gameserver Ansible
======================

This repository contains a collection of Ansible roles to simplify setting up the different components of FAUST's [CTF Gameserver](https://www.ctf-gameserver.org) on multiple individual hosts or a single, shared one. They can be used in your own Ansible playbooks.

What's Included
---------------
* `fausecteam.ctf_gameserver_ansible.checker` performs a basic installation of the [Checker component](https://www.ctf-gameserver.org/checker.html) and configuration of the Checkermaster. You still have to add your own config to check individual services.
* `fausecteam.ctf_gameserver_ansible.controller` installs and configures the [Controller component](https://www.ctf-gameserver.org/controller.html) including scoring.
* `fausecteam.ctf_gameserver_ansible.db_prolog` installs PostgreSQL, creates the required databases, sets their permissions and prepares the main database for initialization through the `web` role. No configuration of the PostgreSQL server is performed, so you probably still have to and fine-tune the settings in "postgresql.conf" and allow remote access in "pg\_hba.conf".
* `fausecteam.ctf_gameserver_ansible.db_epilog` adjusts database permissions once the main database has been initialized through the `web` role.
* `fausecteam.ctf_gameserver_ansible.submission` performs installation and configuration of the [Submission component](https://www.ctf-gameserver.org/flags.html#submission).
* `fausecteam.ctf_gameserver_ansible.vpnstatus` performs installation and configuration of the VPN Status Checker component.
* `fausecteam.ctf_gameserver_ansible.web` installs and configures the [Web component](https://www.ctf-gameserver.org/web.html). It also intializes the main databse (using Django's facilities). Only the raw WSGI interface is provided, so you still have to set up a web and application server (like uwsgi) yourself.

Requirements
------------
The installation is designed to run on Debian GNU/Linux with systemd. At the moment, version Debian 12 ("Bookworm") is our primary target. Ubuntu might work as well, but has not been tested.

The roles do not have any notable requirements on recent Ansible features. They should work with any Ansible version that supports Collections, which is Ansible (or ansible-core) >= 2.9.

It is expected that you build your own Debian packages for CTF Gameserver as described [in the documentation](https://ctf-gameserver.org/installation/#package-build). These must be available under the base URL in the `ctf_gameserver_downloadpath` variable (see below).

All roles expect be run as root user, either through direct root login or using Ansible's [privilege escalation facilities](https://docs.ansible.com/ansible/2.4/become.html). For the `db_prolog` and `db_epilog` roles, you have to make sure that `become`-ing an unprivileged user is possible as [described in the Ansible docs](https://docs.ansible.com/ansible/2.4/become.html#becoming-an-unprivileged-user). From our experience, the easiest option for that is [enabling ACL support](https://help.ubuntu.com/community/FilePermissionsACLs#Enabling_ACLs_in_the_Filesystem) for your file system.

How To Use
----------
### Installation
Either install the Collection through [Ansible Galaxy](https://docs.ansible.com/ansible/2.9/cli/ansible-galaxy.html) by running `ansible-galaxy collection install fausecteam.ctf_gameserver_ansible` or add this repository to your playbook's repository as a Git submodule.

For the latter, e.g. place the submodule at "ansible\_collections/fausecteam/ctf\_gameserver\_ansible" and add the following to your "ansible.cfg":

    [defaults]
    collections_paths = .:~/.ansible/collections:/usr/share/ansible/collections

### Ordering
When using the roles in your own playbook, ordering is crucial. This is regardless of whether the components should run on individual hosts or a shared one.

1. `fausecteam.ctf_gameserver_ansible.db_prolog`: Must run before all other roles, as it creates their databases.
2. `fausecteam.ctf_gameserver_ansible.web`: Initializes the main database, therefore it should be next.
3. `fausecteam.ctf_gameserver_ansible.db_epilog`: Must run after `fausecteam.ctf_gameserver_ansible.web`.
4. `fausecteam.ctf_gameserver_ansible.controller`, `fausecteam.ctf_gameserver_ansible.submission`, `fausecteam.ctf_gameserver_ansible.checker`, and `fausecteam.ctf_gameserver_ansible.vpnstatus`: The ordering between these does not really matter.

### Variables
The roles' behavior can be tuned with various Ansible variables. You can set these [wherever you set variables](https://docs.ansible.com/ansible/2.4/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) for your playbook, for example at the group and host level or even in [Ansible Vault](https://docs.ansible.com/ansible/2.4/vault.html).

Most of the variables have default values, but some do not and are therefore strictly required for you to set. The following is a list of variables which must be set or at least are commonly set. For a list of other options and their defaults, have a look at the "defaults/main.yml" files of the roles.

* All roles
    * `ctf_gameserver_downloadpath`: The base URL under which your Debian packages for CTF Gameserver are available
    * `ctf_gameserver_db_host`: Defaults to "localhost", must at least be changed if the different components run on individual hosts

* `fausecteam.ctf_gameserver_ansible.db_prolog`
    * `ctf_gameserver_db_pass_web`: Password of the web component's database user (name is set through `ctf_gameserver_db_user_web`, "gameserver_web" by default)
    * `ctf_gameserver_db_pass_controller`: Password of the controller component's database user (name is set through `ctf_gameserver_db_user_controller`, "gameserver_controller" by default)
    * `ctf_gameserver_db_pass_submission`: Password of the submission component's database user (name is set through `ctf_gameserver_db_user_submission`, "gameserver_submission" by default)
    * `ctf_gameserver_db_pass_checker`: Password of the checker component's database user (name is set through `ctf_gameserver_db_user_checker`, "gameserver_checker" by default)

* `fausecteam.ctf_gameserver_ansible.web`
    * `ctf_gameserver_db_pass_web`: See above
    * `ctf_gameserver_web_admin_email`: Email address of the admin user to be created (name is set
      through `ctf_gameserver_web_admin_user`, "admin" by default)
    * `ctf_gameserver_web_admin_pass`: Password of the admin user to be created
    * `ctf_gameserver_web_from_email`: The sender address for emails sent by the web component
    * `ctf_gameserver_web_secret_key`: (Ideally) random string
    * `ctf_gameserver_web_https`: Defaults to `false`, change it if your website is available exclusively through HTTPS
    * `ctf_gameserver_web_email_host`: Defaults to "localhost", mailserver for the web component
    * `ctf_gameserver_web_email_use_tls`: Defaults to `false`, change to use STARTTLS for talking to the mailserver
    * `ctf_gameserver_web_timezone`: Defaults to "UTC", change for a different competition timezone

* `fausecteam.ctf_gameserver_ansible.controller`
    * `ctf_gameserver_db_pass_controller`: See above
    * `ctf_gameserver_checker_ippattern`: Defaults to "10.66.%d.2", adjust it if you use a different competition network

* `fausecteam.ctf_gameserver_ansible.submission`
    * `ctf_gameserver_db_pass_submission`: See above
    * `ctf_gameserver_flag_secret`: Secret for the flags's HMAC, i.e. a random byte-string in Base-64 format
    * `ctf_gameserver_submission_listen_host`: Defaults to "localhost", must be changed to listen on another IP address or hostname
    * `ctf_gameserver_submission_listen_ports`: A list of ports to listen on, defaults to the single port 6666

* `fausecteam.ctf_gameserver_ansible.checker`
    * `ctf_gameserver_db_pass_checker`: See above
    * `ctf_gameserver_flag_secret`: See above

* `fausecteam.ctf_gameserver_ansible.vpnstatus`
    * `ctf_gameserver_db_pass_vpnstatus`: See above
    * `ctf_gameserver_vpnstatus_wireguard_ifpattern`: Optional, (old-style) Python formatstring for building a team's Wireguard interface, e.g. "wg%d"
    * `ctf_gameserver_vpnstatus_gateway_ippattern`: Optional, (old-style) Python formatstring for building a team's gateway IP, e.g. "10.66.%d.1"
    * `ctf_gameserver_vpnstatus_demo_ippattern`: Optional, (old-style) Python formatstring for building a team's demo Vulnbox IP
    * `ctf_gameserver_vpnstatus_demo_serviceport`: Optional, TCP port for service checks on demo Vulnboxes
    * `ctf_gameserver_vpnstatus_vulnbox_ippattern`: Optional, (old-style) Python formatstring for building a team's Vulnbox IP
    * `ctf_gameserver_vpnstatus_vulnbox_serviceport`: Optional, TCP port for service checks on Vulnboxes

License
-------
The contents of this repository are released under the ISC License.

Internal Note: Releasing this Collection
----------------------------------------
As a maintainer of this Collection, perform the following steps to release it to Ansible Galaxy:

1. Update `changelogs/changelog.yaml`
2. Update `version` field in `galaxy.yml`
3. Commit these changes
4. Run `ansible-galaxy collection build`
5. Run `ansible-galaxy collection publish <tarball>`
6. Run `git tag -a -s <version>`
