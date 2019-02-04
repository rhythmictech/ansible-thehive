# ansible-thehive

## Description

Deploys and configures [TheHive](https://thehive-project.org), an excellent
open source incident response tool. It installs based off the RPM and can
optionally pre-seed the ElasticSearch index, eliminating some of the annoying
manual steps in getting TheHive running.

You will need to install ElasticSearch separately. The role is tested with the
[Elastic-provided ansible role](https://github.com/elastic/ansible-elasticsearch).
Sample configuration is included in the documentation.

This should cover most use cases of TheHive, but PRs and suggested improvements
are welcome.

## Requirements

* Ansible 2.0 or higher
* CentOS 7
* ElasticSearch 5.x

## Usage

This is recommended to be installed on a dedicated server, though both ElasticSearch
and Cortex can safely be installed together with TheHive. An optional Nginx proxy
is enabled by default, and support is available for Vouch and LDAP authentication.
If using delegated authentication, it is important to correctly set a seed user
that you can log in as.

ElasticSearch must be installed and running already. This role is tested using the
ansible-elasticsearch role, which can be imported from Ansible Galaxy.

The following vars are recommended:

```yaml
es_instance_name: "thehive"
es_version: 5.6.14
es_major_version: 5.x
es_data_dirs:
  - "/data/es"
es_config:
  node.name: "thehive"
  cluster.name: "thehive"
  node.data: true
  node.master: true
  script.inline: on
  thread_pool.index.queue_size: 100000
  thread_pool.search.queue_size: 100000
  thread_pool.bulk.queue_size: 100000
es_scripts: true
es_templates: false
es_version_lock: false
es_heap_size: 1g
es_xpack_features: ["alerting","monitoring"]
```

Note that ElasticSearch 6.x is not supported by TheHive. Currently the master
branch of the ansible-elasticsearch module supports 5.x.

The following vars must be set at a minimum:

* thehive_url (fqdn where thehive will be accessible)
* thehive_crypto_secret (see `defaults/main.yml` for instructions on how to generate this)

A sample common configuration that automatically seeds TheHive and uses LDAP authentication
and Cortex is included below:

```yaml
thehive_url: "thehive.corp"
thehive_seed_initial_username: "admin"

thehive_http_addr: "127.0.0.1"

thehive_crypto_secret: "..."

thehive_auth_ldap:
  enabled: true
  servers: ["ldapserver.corp:636"]
  use_ssl: true
  bind_dn: "bind_dn"
  bind_pw: "bind_pw"
  search_base: "dc=corp"
  username_attribute: "sAMAccountName"
}

thehive_cortex_servers:
  cortex:
    url: "http://127.0.0.1:9001/"
    key: "..."

```

## Vouch Authentication
This role supports authentication through a Vouch (formerly known as Lasso) proxy.
This allows you to do OAUTH authentication through providers such as Okta.

When using Vouch, it is critical to set ```thehive_http_addr``` to 127.0.0.1.
Because Vouch uses cookies to communicate authentication information back to the
application, you must place both your Vouch proxy and TheHive site under a common
domain name (e.g., vouch.corp and thehive.corp).

## Role Variables

```yaml
# Whether or not the TheHive RPM repo should be installed.
# This is generally what you want, unless you are using your own RPM repo.
thehive_install_repo: true

# TheHive version to lock and install
thehive_version: 3.2.1

# Note that the mappings and seed data are dependent on the schema version.
# If you are installing a version of TheHive that uses a different index name,
# the mappings and data files need to be updated.
thehive_index: thehive_14

# TheHive URL.
thehive_url: localhost

# Wheteher or not an nginx instance should be installed as a proxy
thehive_install_nginx: true

# Whether or not to configure nginx proxy
thehive_configure_nginx: true

# Referenced files will be included in each nginx server config
thehive_nginx_includes: []

# Optionally use SSL with Nginx
thehive_nginx_ssl:
  enabled: false
  certificate: ""
  key: ""
  #cabundle: provide if using a bundle

# The port TheHive will listen to. This var can be changed even when using
# the nginx proxy.
thehive_http_port: 9000

# IP address TheHive should bind to. In general, this can be left as is. However,
# this must be set to 127.0.0.1 when authenticating through a proxy
thehive_http_addr: "0.0.0.0"

# Mandatory. Generate a key like this:
# cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1
thehive_secret: ""

# By default, TheHive requires manual steps to configure.
# You can optionally load a pre-configured mapping and seed data, which makes
# TheHive immediately usable out of the box.
thehive_load_seed_data: true

# Name of the initial user to create. Note if you are using vouch or LDAP for
# authentication, you must set this to a valid username in your directory.
# TheHive does not create users on first logon.
thehive_seed_initial_username: "admin"

# Optionally use Vouch authentication (e.g., for Google Authentication, Okta, etc)
thehive_auth_vouch:
  enabled: false
  url: ""
  logon_header: THEHIVE_USER

# Optionally use LDAP authentication.
thehive_auth_ldap:
  enabled: false
  servers: []
  use_ssl: ""
  bind_dn: ""
  bind_pw: ""
  search_base: ""
  username_attribute: "cn"

# ElasticSearch configuration. If using recommended ES configuration, this
# does not need to be changed.
thehive_es:
  index: thehive
  cluster: thehive
  endpoint: 127.0.0.1:9300

# Packages that will be installed with TheHive
thehive_packages:
  - java-1.8.0-openjdk
  - python-pip
  - unzip
  - git
  - thehive-{{ thehive_version }}

# Packages that will be installed if the nginx proxy is used.
# libsemanage-python is necessary for selinux.
thehive_nginx_packages:
  - nginx
  - libsemanage-python
```
