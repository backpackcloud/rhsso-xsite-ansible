# RH-SSO X-Site Replication Playbook

## Inventory Grouping

The playbooks expect the following inventory groups:

- `rhsso`: which nodes are part of the RH-SSO cluster, regardless of which Data Center they belong to.
- `datagrid`: which nodes are part of the Data Grid cluster, regardless of which Data Center they belong to.

In addition to the grouping, a host variable `site` is expected in order to group the nodes across the different
Data Centers. The `site` value should point to the data center group which the host belongs to.

Example:

```yaml
all:
  children:

    EU:
      vars:
        site: EU
        
      hosts:
        rhsso-eu1.example.com:
        rhsso-eu2.example.com:

        rhdg-eu1.example.com:
        rhdg-eu2.example.com:

    US:
      vars:
        site: US

      hosts:
        rhsso-us1.example.com:
        rhsso-us2.example.com:

        rhdg-us1.example.com:
        rhdg-us2.example.com:

    datagrid:
      hosts:
        rhdg-eu[1:2].example.com:
        rhdg-us[1:2].example.com:

    rhsso:
      hosts:
        rhsso-eu[1:2].example.com:
        rhsso-us[1:2].example.com:
```

## Defining Where Packages are Located

There are two ways of defining how to transfer the packages and patches to the servers: local transfer and url fetch.
They are controlled by a suffix, whereas the product they refer to is controlled by a prefix.

For Data Grid, use the prefix `datagrid`, for RH-SSO use the prefix `rhsso`.

For local transfer, use the suffix `path`, for url fetch, use the suffix, `url`.

The patches are applied using the same approach, so if you're using local transfer for the packages, you would have to
use the same for the patches.

Examples:

```yaml
datagrid_package_path: .bin/redhat-datagrid-8.2.1-server.zip

rhsso_package_path: .bin/rh-sso-7.4.0.zip
rhsso_patches:
  - path: .bin/rh-sso-7.4.8-patch.zip
    name: rh-sso-7.4.8-patch.zip
```

```yaml
datagrid_package_url: https://storage.example.com/files/redhat/rhdg/redhat-datagrid-8.2.1-server.zip

rhsso_package_url: https://storage.example.com/files/redhat/rhsso/rh-sso-7.4.0.zip
rhsso_patches:
  - path: https://storage.example.com/files/redhat/rhsso/rh-sso-7.4.8-patch.zip
    name: rh-sso-7.4.8-patch.zip
```

## Tags

The playbooks are organized using tags for each phase of the installation:

- `system`: used for all operating system related tasks, such as creating users, installing packages and configuring 
            firewall
- `install`: used for the product installation tasks, like transferring files, unpacking and applying patches
- `configure`: used for configuring the product after installation

The most useful tag is `configure`. It allows to reconfigure the product without doing the previous steps once the
product is installed once.

## Idempotence

The playbooks are idempotent, so there is no need to delete anything before running the whole installation process
again.

## Data Grid

The role `datagrid` is responsible for the installation and configuration of Data Grid to act as a remote cache for
RH-SSO. It uses TCPPING as the discovery mechanism together with the RELAY2 for the Cross Site Replication.

The expected variables to be supplied are:

`console_users`: a map holding pairs of username/password to be added to the Data Grid. By a Data Grid convention, any
                 user named `admin` will have admin privileges. This should be supplied via a vault file to avoid
                 storing credentials in plain text.

`bind_address`: which IP address to bind the Data Grid service. Defaults to `ansible_ssh_host` value.

`jgroups_bind_address`: which IP address to bind the JGroups service for clustering. Defaults to ´ansible_ssh_host`
                        value.

`nodename`: the node name to use across the configuration files. It needs to be resolved for all nodes. Defaults to
            `inventory_hostname`.

`max_number_of_masters`: defines the maximum number of masters in the Data Grid topology. It's a good practice to allow
                         all nodes to be masters, but this configuration is provided in case it needs to be adjusted.
                         Defaults to the number of Data Grid nodes defined in the `datagrid` group.

## RH-SSO

The role `rhsso` is responsible for the installation and configuration of Red Hat Single Sign-On using the previous
Data Grid cluster as a remote cache in order to implement Cross Site Data Center Replication. It uses TCPPING as the
discovery mechanism for the cluster formation and includes the MariaDB JDBC driver.

The expected variables to be supplied are:

`bind_address`: which IP address to bind the Data Grid service. Defaults to `ansible_ssh_host` value.

`jgroups_bind_address`: which IP address to bind the JGroups service for clustering. Defaults to ´ansible_ssh_host`
value.

`nodename`: the node name to use across the configuration files. It needs to be resolved for all nodes. Defaults to
`inventory_hostname`.

`jvm`: a map with parameters for the JVM, where `heap` is the Heap size and `metaspace` is the Metaspace size. Defaults
       to:

```yaml
jvm:
  heap: 1303m
  metaspace: 256m
```

`admin`: the credentials for the admin user. This should be supplied using vault to avoid storing credentials on plain
         text files. It can be manually added later if not supplied in the inventory.

Example:

```yaml
admin:
  username: admin
  password: super-strong-password-that-you-would-never-guess
```

`cache_owners`: the number of cache owners for the RH-SSO infrastructure. A good rule of thumb is to use around 3 owners
                to not overload the network with the cache replication. Defaults to the number of nodes or 3, whichever
                is lower.

`rhsso_datagrid_connection`: the connection to the Data Grid node. It's not necessary to provide all nodes as the whole cluster view is
                             passed during the connection establishment. It expects a `host`, `username`, and `password` and can also be
                             provided as part of a vault file.

Example:

```yaml
rhsso_datagrid_connection:
  host: rhdg-eu1.example.com
  username: developer
  password: developer
```

`rhsso_datasource`: defines the connection to the Database. It expects a `connection_url`, `username` and `password` and can
                    also be provided as part of a vault file.

Example:

```yaml
rhsso_datasource:
  connection_url: jdbc:mariadb://mariadb-eu.example.com:3306/rhsso
  username: rhsso
  password: rhsso 
```

The playbook contains fixes for some known bugs present in earlier versions of RH-SSO. Those fixes don't impact in the
product and can be disabled by skipping the tag `fix`. This is useful in order to test if the version used already fix
the issues. The issues are described at:

- https://access.redhat.com/solutions/4243861
- https://access.redhat.com/solutions/4303601
