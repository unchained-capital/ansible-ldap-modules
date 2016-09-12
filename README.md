# Ansible LDAP Modules

For whatever reasons, Ansible doesn't include modules to manipulate an
LDAP server in its [core](http://docs.ansible.com/ansible/modules_by_category.html).

Peter Sagerson implemented a pair of modules and has
[attempted to get them into Ansible](http://grokbase.com/t/gg/ansible-devel/148892tek3/ldap-modules),
but no luck yet.  He's instead hosted his code on
[Bitbucket](https://bitbucket.org/psagers/ansible-ldap).

This repository is a fork of his original work (with some cosmetic
improvements) hosted on GitHub.

# Installation

You'll need to clone this repository somewhere into your Ansible (use
a git submodule).

```
$ git clone https://github.com/unchained-capital/ansible-ldap-modules
```

The `python-ldap` library is required on the Ansible controller node
for these modules to load.  Install it with

```
$ cd ansible-ldap-modules
$ sudo pip -r requirements.txt
```

You may need to install some system dependencies first:

```
$ sudo apt-get install python-dev libsasl2-dev libldap2-dev libssl-dev
```

Once you have `python-ldap` installed, you'll need to link the
executable files in this repository into Ansible's library path.  The
simplest way to do this is to create a `library` folder at the
top-level of your Ansible repository and create symlinks within it to
the module files in this repository:

```
$ mkdir -p library
$ ln -s ansible-ldap-modules/ldap_entry library/ldap_entry
$ ln -s ansible-ldap-modules/ldap_attr  library/ldap_attr
```

You can also explicitly set the `ANSIBLE_LIBRARY` environment variable
or the `library` entry within the `defaults` section of your
`ansible.cfg` to include this repository's directory.

# Usage

(copied from psager's [original README](https://bitbucket.org/psagers/ansible-ldap/src/ca4c0025358cdeba33e9ef0369af3430bf4812ad/README.md))

This project contains a pair of Ansible modules for manipulating an
LDAP directory. `ldap_entry` can be used to ensure that an entire
entry exists and `ldap_attr` can be used to ensure the values of an
entry's attributes.

Regrettably, Ansible does not have any sensible mechanism for
packaging and distributing third-party modules with rendered
documentation and runnable unit tests. The LDAP modules do have
complete documentation strings embedded.
