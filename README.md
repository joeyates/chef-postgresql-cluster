# chef-postgresql-replication

A Chef cookbook which provides recipes for managing groups of
servers with interdependent PostgreSQL clusters set up
as masters and replication slaves.

# Directives

This cookbook manages groups of clusters across a number of servers.

For this reason all directives have the required parameter `group`.

Behaviour changes when deploying to a server which is master or slave.

## `postgresql_replication_group`

This directive declares the group and indicates the PostgreSQL version
to use for all clusters on all servers that are part of the group.

Actions:

* `:create`

Attributes:

* `group` - (name attribute)
* `version` - (required) `"9.4"`. The version of PostgreSQL to install
* `replication_user` - (required) the database user that will carry out
  replication. This user should be created on the master (see
  `postgresql_replication_user`
  On the master, the necessary access entries are set in `pg_hba.conf`
* `max_connections` - (default: 100). As all servers in the group must
  have matching values for `max_connections`, this value overrides
  whatever is set for the individual nodes

Example:

```ruby
postgresql_replication_group "my_group" do
  version "9.4"
  replication_user "bob"
end
```

## `postgresql_replication_master`

Indicates which host is master for a given group.

There should be **exactly one** master for each group.

If more than one is declared, an error is raised.

If the Chef run is deploying to the indicated host, all slave data (see below)
is used to:
* create replication users
* give them replication access (by creating the replication user and adding
  an entry for the user to `pg_hba.conf`)

Actions:

* `:create`

Attributes:

* `hostname` - (name attribute)
* `group` - (required)

Example:

```ruby
postgresql_replication_master "foo.example.com" do
  group "my_group"
end
```

N.B. A separate `postgresql_replication_node` directive should exist to set up the
PostgreSQL node itself.

## `postgresql_replication_slave`

If the Chef run is deploying to the indicated host, this directive initiates
replication.

N.B. A separate `postgresql_replication_node` directive should exist to set up the
PostgreSQL node itself.

Actions:

* `:create`

Attributes:

* `hostname` - (name attribute)
* `group` - (required)
* `ip` - (required) The IP address of the host
* `master` - (default: the hostname declared in `postgresql_replication_master`)
  Allows a slave to replicate from another replication slave in the group.
* `hot_standby` - `true`|`false` (default: `true`) Indicates if the slave should
  allow read-only connections while replicating (by setting `hot_standby` to
  `on`). If the attributes is set to false, and master is set too `true` - an error
  is raised.

Example:

```ruby
postgresql_replication_slave "bar.example.com" do
  group "my_group"
  ip "200.200.200.200"
end
```

## `postgresql_replication_node`

This is the actual configuration for the PostgreSQL cluster on a host.

Actions:

* `:create`
* `:delete`

Attributes:

* `hostname` - (name attribute)
* `group` - (required)
* `pg_hba` - entries for `pg_hba.conf`
* `conf` - entries for `postgresql.conf`

When a node is to be used as a master - either the main master for the group,
or as a slave which is used as master by another slave - the cluster is set up
to:
* write WAL logs,
* allow access to the `replication_user` from all slaves.
When a slave node is to be used as master for other slaves, the `hot_standby`
setting is always set to `on`.

Example:

```ruby
postgresql_replication_node "foo.example.com" do
  group "my_group"
  pg_hba ...
  conf ...
end
```

## `postgresql_replication_user`

N.B. This directive creates *normal database users*. To indicate which
user should be set up to handle replication set `replication_user` in the
`postgresql_replication_group` directive.

This directive is ignored unless deploying to a master.

User resources do nothing if a `postgresql_replication_master` for the
group has not been declared.

Actions:

* `:create` (default)
* `:delete`

Attributes:

* `username` - (name attribute)
* `group` - (required)
* `password`

Example:

```ruby
postgresql_replication_user "bob" do
  group "my_group"
  password "secret"
end
```

# `postgresql_replication_database`

N.B. This directive creates *normal databases*

Actions:

* `:create` (default)
* `:delete`

Attributes:

* `database` - (name attribute)
* `group` - (required)
* `owner` - (default: "postgres")
* `encoding`
* `template`

Example:

```ruby
postgresql_replication_database "my_database" do
  group "my_group"
  owner "bob"
end
```

# Failover

## What to change

* choose a slave as the new master and change that host's directive from
  `postgresql_replication_slave` to `postgresql_replication_master`,
* change the old master's setup:
  * if the master is to become a slave, change its directive to
    `postgresql_replication_slave` and set the necessary attributes (e.g.
    `username),
  * if the master is to be removed, delete its `postgresql_replication_master`
    and change its `postgresql_replication_node` action to `:delete`.
* deploy to the new master,
* if necessary switch DNS entries which need to point to the master
  to point to the new master,
* deploy to other slaves (if possible, they will simply continue replicating
  from the new master, ootherwise, replication will be re-initialized from
  the new master),
* deploy to the master, if it is accessible.

## What the cookbook does

on an old slave when switching to `master`:

* notices the cluster exists and is acting as replication slave,
* applies any configuration changes and restarts if necessary,
* touches the failover flag file to halt recovery and switch to being
  master.

on the old master when switching to `down`:

* stops the cluster,
* applies any configuration changes.

on the old master when switching to `slave`:

* notices the cluster exists and is acting as a master,
* stops the cluster,
* configures the cluster to be a slave,
* initiates replication,
* starts the cluster.

# Similar cookbooks

* https://github.com/express42-cookbooks/postgresql
  * Doesn't handle 9.4 yet,
  * Always does shared memory expansion?
  * Each cluster is set up separately. No management of failover.
* https://github.com/Lispython/pg_cluster-cookbook
