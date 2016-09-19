# Ansible LDAP Modules

For whatever reasons, Ansible doesn't include modules to manipulate an
LDAP server in its
[core](http://docs.ansible.com/ansible/modules_by_category.html).

Peter Sagerson implemented a pair of modules and has
[attempted to get them into Ansible](http://grokbase.com/t/gg/ansible-devel/148892tek3/ldap-modules),
but no luck yet.  He's instead hosted his code on
[Bitbucket](https://bitbucket.org/psagers/ansible-ldap).

This repository is a fork of his original work with some additions
(and cosmetic improvements) hosted on GitHub.

# Installation


The `python-ldap` library is required on whatever host is executing
these modules.  This can be confusing if you use both
`local_action`-type tasks and tasks which run on remote hosts and both
need to talk do an LDAP server -- you'll need to install `python-ldap`
both locally on your controller as well as remotely on the hosts you
run the tasks on.

In addition, you'll need to clone this repository itself to your
controller and put it somewhere `ansible` can find it.


## python-ldap Installation

Install it locally via the following commands:

```
$ cd ansible-ldap-modules
$ sudo pip -r requirements.txt
```

You may need to install some system dependencies first:

```
$ sudo apt-get install python-dev libsasl2-dev libldap2-dev libssl-dev
```

For remote hosts, this can be automated via Ansible:

```yaml
# in prepare_remote_ldap_host.yml, for example
- name: Install dependencies
  apt: name="{{ item }}" state=present
  with_items:
    - python-pip
    - libsasl2-dev
    - libldap2-dev
    - libssl-dev

- name: Upgrade pip
  command: pip install --upgrade pip

- name: Install python-ldap
  pip: name=python-ldap state=present
```

## Modules Installation

You'll need to clone this repository somewhere on your controller
machine so that `ansible` can find it.

```
$ git clone https://github.com/unchained-capital/ansible-ldap-modules
```

Once you have `python-ldap` installed, you'll need to link the
executable files in this repository into Ansible's library path.  The
simplest way to do this is to create a `library` folder at the
top-level of your playbook repository and create symlinks within it to
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

Remember, if you run these tasks locally, you need `python-ldap`
installed locally.  If you run them on a remote machine, you need
`python-ldap` installed on that remote machine.

## Shared Behavior

All the modules in this repository share some behaviors.  This section
describes those behaviors.  The next section describes each module in
detail.

### Specifying an LDAP Server

Here is a simple example of creating an entry (more details on this
below):

```yaml
- hosts: server0
  tasks:
  - name: Ensure an LDAP entry exists for ou=People
    ldap_entry:
      dn: "ou=People,dc=example,dc=com"
      ou: People
      objectClass: organizationalUnit
      description: Getting together and having a good time.
```

The target host of an LDAP operation is assumed to be the same host as
the Ansible task is executing so the above task would attempt to talk
to LDAP server running on `server0` listenting on port 389.  This example:

```yaml
- hosts: server0
  tasks:
  - name: Ensure an LDAP entry exists for ou=People
    ldap_entry:
	  server_uri: ldapi://server1/
      dn: "ou=People,dc=example,dc=com"
      ou: People
      objectClass: organizationalUnit
      description: Getting together and having a good time.
```

would target an LDAP server at `server1`.

### Choosing Credentials

Without any credentials (as in the above examples), the request will
be made via a SASL `EXTERNAL` bind (similar to `ldapadd ... -Y
EXTERNAL ...` on the command-line).

Credentials can be specified as well:


```yaml
- hosts: server0
  tasks:
  - name: Ensure an LDAP entry exists for ou=People
    ldap_entry:
	  bind_dn: "cn=admin,dc=example,dc=com"
	  bind_pw: "password"
      dn: "ou=People,dc=example,dc=com"
      ou: People
      objectClass: organizationalUnit
      description: Getting together and having a good time.
```

## Modules

This repository provides four different modules:

### ldap_entry

Ensures that an entry with a given `dn` exists/doesn't exist.  If
missing, it creates it.  It present it does nothing.  In particular,
even if the entry has different attributes from those specified in
Ansible, `ldap_entry` does nothing.  You can, however, use
`ldap_entry` to remove LDAP entries if you specify `state=absent`.

#### Creating an entry

```yaml
- name: Ensure an LDAP entry exists for ou=People
  ldap_entry:
    dn: "ou=People,dc=example,dc=com"
	ou: People
	objectClass: organizationalUnit
	description: Getting together and having a good time.
```

If the `ou=People,dc=example,dc=com` entry has its `description` field
changed, this task will not update it.  It's therefore best to only
specify the minimal number of fields required to successfully create
the entity, given its LDAP `objectClass` values and later use
`ldap_attr` to explicitly declare the expected state for each
attribute individually.  Or use `ldap_upsert`.

#### Removing an entry

```yaml
- name: Ensure an LDAP entry exists for ou=People
  ldap_entry:
    dn: "ou=People,dc=example,dc=com"
    state: absent
```

### ldap_attr

Ensures that an attribute with a given value exists/doesn't exist for
a given entry.  If the entry does not exist, it throws an error.  If
the doesn't exist, it creates it.  If it exists but has a different
value, it updates it.  You can use `ldap_attr` to remove LDAP
attributes if you specify `state=absent`.

#### Specifying an attribute exactly

Here's a simple example.

```yaml
- name: Ensure description is correct for ou=People
  ldap_attr:
    dn: "ou=People,dc=example,dc=com"
	name:  description
	state:  exact
	values: Getting together and having a good time.

- name: Ensure members are correct for cn=Admins,ou=Groups
  ldap_attr:
    dn: "cn=Admins,ou=Groups,dc=example,dc=com"
	name:   member
	state:  exact
	values:
	  - cn=joe,ou=People,dc=example,dc=com
	  - cn=bob,ou=People,dc=example,dc=com
```
This is a lot of work for many entries with many attributes.  Consider
`ldap_upsert` in these cases.

#### Appending to an attribute

```yaml
- name: Ensure members are present in cn=Admins,ou=Groups
  ldap_attr:
    dn: "cn=Admins,ou=Groups,dc=example,dc=com"
	name:   member
	state:  present
	values:
	  - cn=joe,ou=People,dc=example,dc=com
	  - cn=bob,ou=People,dc=example,dc=com
```

The user `cn=mike,ou=People,dc=example,dc=com` could still be a
`member` of `cn=Admins,ou=Groups,dc=example,dc=com` after this task
runs.

#### Removing an attribute

```yaml
- name: Ensure user's password is not set
  ldap_attr:
    dn: "cn=joe,ou=People,dc=example,dc=com"
	state: absent
	name:  userPassword
```

### ldap_upsert

Ensures both that an entry exists as well as that its attributs have
particular values.  You cannot use `ldap_upsert` to remove entries or
their attributes, but it useful when creating lots of entities with
their attributes.

```yaml

- name: Create a user
  ldap_upsert:
    dn:    "uid=joe1234,ou=People,dc=example,dc=com"
	objectClass:
	  - account
	  - posixAccount
	cn:    "Joe Smith"
	gn:    "Joe"
	sn:    "Smith"
	uid:   "joe1234"
	homeDirectory: "/home/joe1234"
	userPassword: "..."
```

### ldap_search

Perform an LDAP search.  Useful in combination with Ansible's
`register` keyword.

This example performs a task for each LDAP user `account`:

```yaml
- name: 
  ldap_search:
    base: "ou=People,dc=example,dc=com"
	scope: onelevel
	filter: "(objectClass=account)"
  register: ldap_user_search

- name: View search results
  debug: var=ldap_user_search

- name: Do something for each user
  # ...
  with_items: "{{ ldap_user_search.results }}"
```

