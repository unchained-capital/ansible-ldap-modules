#!/usr/bin/env python

from traceback import format_exc

import ldap
import ldap.modlist
import ldap.sasl


DOCUMENTATION = """
---
module: ldap_search
short_description: Return the results of an LDAP search.
description:
    - Return the results of an LDAP search.  Use in combination with
      Ansible's 'register' statement.

notes: []
version_added: null
author: Dhruv Bansal
requirements:
    - python-ldap
options:
    base:
        required: true
        description:
            - The base to search from.
    scope:
        required: false
        choices: [base, onelevel, subordinate, children]
        default: base
        description:
            - The LDAP scope to use when searching.
    filter:
        required: false
        default: '(objectClass=*)'
        description:
            - The filter to apply to the search.
    attrs:
        required: false
        default: none
        description:
            - A list of attrs to limit the results to.  Can be an
              actual list or just a comma-separated string.
    schema:
        required: false
        default: false
        description:
            - Return the full attribute schema of entries, not their
              attribute values.  Overrides C(attrs) when given.
    server_uri:
        required: false
        default: ldapi:///
        description:
            - A URI to the LDAP server. The default value lets the underlying
              LDAP client library look for a UNIX domain socket in its default
              location.
    start_tls:
        required: false
        default: false
        description:
            - If true, we'll use the START_TLS LDAP extension.
    bind_dn:
        required: false
        description:
            - A DN to bind with. If this is omitted, we'll try a SASL bind with
              the EXTERNAL mechanism. If this is blank, we'll use an anonymous
              bind.
    bind_pw:
        required: false
        description:
            - The password to use with C(bind_dn).
"""


EXAMPLES = """
# Return all entries within the 'groups' organizational unit.
- ldap_search: base='ou=groups,dc=example,dc=com'
  register: ldap_groups
  sudo: true

# Return GIDs for all groups
- ldap_entry: base='ou=groups,dc=example,dc=com' scope=onelevel attrs="gidNumber"
  register: ldap_group_gids
  sudo: true
"""

def main():
    module = AnsibleModule(
        argument_spec={
            'base': dict(required=True),
            'scope': dict(default='base', choices=['base', 'onelevel', 'subordinate', 'children']),
            'filter': dict(default='(objectClass=*)'),
            'attrs': dict(default=None),
            'schema': dict(default=False, choices=(list(BOOLEANS)+['True', True, 'False', False])),
            'server_uri': dict(default='ldapi:///'),
            'start_tls': dict(default='false', choices=(list(BOOLEANS)+['True', True, 'False', False])),
            'bind_dn': dict(default=None),
            'bind_pw': dict(default='', no_log=True),
        },
        check_invalid_arguments=False,
        supports_check_mode=False,
    )

    try:
        LdapSearch(module).main()
    except ldap.LDAPError, e:
        module.fail_json(msg=str(e), exc=format_exc())


class LdapSearch(object):
    _connection = None

    def __init__(self, module):
        self.module = module

        # python-ldap doesn't understand unicode strings. Parameters that are
        # just going to get passed to python-ldap APIs are stored as utf-8.
        self.base = self._utf8_param('base')
        self.filterstr = self._utf8_param('filter')
        self.server_uri = self.module.params['server_uri']
        self.start_tls = self.module.boolean(self.module.params['start_tls'])
        self.bind_dn = self._utf8_param('bind_dn')
        self.bind_pw = self._utf8_param('bind_pw')
        self.attrlist = []

        self._load_scope()
        self._load_attrs()
        self._load_schema()

        # if (self.state == 'present') and ('objectClass' not in self.attrs):
        #     self.module.fail_json(msg="When state=present, at least one objectClass must be provided")

    def _utf8_param(self, name):
        return self._force_utf8(self.module.params[name])

    def _load_schema(self):
        self.schema = self.module.boolean(self.module.params['schema'])
        if self.schema:
            self.attrsonly = 1
        else:
            self.attrsonly = 0

    def _load_scope(self):
        scope = self.module.params['scope']
        if   scope == 'base':        self.scope = ldap.SCOPE_BASE
        elif scope == 'onelevel':    self.scope = ldap.SCOPE_ONELEVEL
        elif scope == 'subordinate': self.scope = ldap.SCOPE_SUBORDINATE
        elif scope == 'children':    self.scope = ldap.SCOPE_SUBTREE
        else:
            self.module.fail_json(msg="scope must be one of: base, onelevel, subordinate, children")

    def _load_attrs(self):
        if self.module.params['attrs'] is None:
            self.attrlist = None
        else:
            attrs = self._load_attr_values(self.module.params['attrs'])
            if len(attrs) > 0:
                self.attrlist = attrs
            else:
                self.attrlist = None

    def _load_attr_values(self, raw):
        if isinstance(raw, basestring):
            values = raw.split(',')
        else:
            values = raw

        if not (isinstance(values, list) and all(isinstance(value, basestring) for value in values)):
            self.module.fail_json(msg="attrs must be a string or list of strings.")

        return map(self._force_utf8, values)

    def _force_utf8(self, value):
        """ If value is unicode, encode to utf-8. """
        if isinstance(value, unicode):
            value = value.encode('utf-8')

        return value

    def main(self):
        results = self.perform_search()
        self.module.exit_json(changed=True, results=results)

    def perform_search(self):
        try:
            results = self.connection.search_s(self.base, self.scope, filterstr=self.filterstr, attrlist=self.attrlist, attrsonly=self.attrsonly)
            if self.schema:
                return [dict(dn=result[0],attrs=result[1].keys()) for result in results]
            else:
                return [self._extract_entry(result[0], result[1]) for result in results]
        except ldap.NO_SUCH_OBJECT:
            self.module.fail_json(msg="Base not found: {}".format(self.base))

    def _extract_entry(self, dn, attrs):
        extracted = {'dn': dn}
        for attr, val in attrs.iteritems():
            if len(val) == 1:
                extracted[attr] = val[0]
            else:
                extracted[attr] = val
        return extracted

    #
    # LDAP Connection
    #

    @property
    def connection(self):
        """ An authenticated connection to the LDAP server (cached). """
        if self._connection is None:
            self._connection = self._connect_to_ldap()

        return self._connection

    def _connect_to_ldap(self):
        connection = ldap.initialize(self.server_uri)

        if self.start_tls:
            connection.start_tls_s()

        if self.bind_dn is not None:
            connection.simple_bind_s(self.bind_dn, self.bind_pw)
        else:
            connection.sasl_interactive_bind_s('', ldap.sasl.external())

        return connection


from ansible.module_utils.basic import *  # noqa
main()
