##### Note
This ansible role was originally templated from [isaackehle](https://github.com/isaackehle)'s work [here](https://github.com/isaackehle/ansible-mongodb).
The reason for a different role is that this one is intended to be more light weight and thus prune's some functionalities from the original which some users may still need. These include:
- [x509 security](https://www.mongodb.com/docs/manual/core/security-x.509/)

This set up implements authentication using a mongodb keyfile and is thus ideal to have the servers runing in a VPC.

# Ansible Role - mongodb

Configure the components of a MongoDB Cluster
Available on [Ansible Galaxy](https://galaxy.ansible.com/123mwanjemike/ansible_mongodb)

## Variables
Required definitions are as follows:

```yaml
cfg_server:
  name: "my-cfg" # (Required)
  group: "my-cfg-servers" # (Required) Always pass in the group id used for the config servers

replica_set:
  name: "my-cfg" # name of the replica set for the config server (prefix of fqdn)
  group: "my-cfg-servers" # group name for all servers in the replica set
```


### Roadmap
**May 2023**
- [x] Creates admin user
- [ ] Creates the normal user

**June 2023**
- [x] Initiates the config replicaset(s)
- [x] Can initiate a shard replicaset(s)
- [ ] Adds the shard replicaset(s)

Host Definitions typically contain the following:

### Replica Set Only
```yaml
cluster_role: "replicaSet"
```

### Router Server
```yaml
cluster_role: "router"
```

### Config Server
```yaml
cluster_role: "config"
replica_set:
  name: "my-cfg" # name of the replica set for the config server (prefix of fqdn)
  group: "my-cfg-servers" # group name for all servers in the replica set
```

### Shard Server
```yaml
cluster_role: "shard"
replica_set:
  name: "db-data" # name of the replica set for the shard server (prefix of fqdn)
  group: "db-data-servers" # group name for all servers in the replica set
```

## Tags/Flags
You can use a system of flags and tags that allow the calling playbook to specify which roles are run.
For example:

```bash
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['install']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['save_config']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['prepare_members']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['init_replica']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['add_shard']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['create_database']}"
```

## Flags and Variables
| Flag                 | Purpose                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| install_mongo        | Installs mongo packages                                                          |
| start_mongo          | Starts mongo database and/or server                                              |
| stop_mongo           | Stop mongo database and/or server                                                |
| configure_mongo      | Configure mongo                                                                  |
| prepare_members      | Prepares the mongodb sharded cluster members                                     |
| init_replica         | Initialize the replica set configuration                                         |
| create_admin         | Creates the admin user                                                           |
| add_shard            | Add a replica set of a shard server to the cluster of shard servers              |
| create_database      | Do an initial database creation, with username and password                      |
| clear_logs           | Clears all the sharded cluster logs                                              |

```yaml
vars:
  flags: ["install"]
  new_shard:
    name: # Name of the replica set to add to the config server
    server: # One of the members of the new replicate set to add
```

###### Sample playbook
```yaml
- hosts: all
  vars:
    auth_db: ""
    adminUser: ""
    adminPass: ""
    tgt_db: ""
    userName: ""
    userPass: ""
    roles: ["readWrite", "userAdmin"]

    # For when initializing the replica set
    adminUser: ''
    adminPass: ''

  roles:
    - { role: 123mwanjemike.mongodb, flags: ['install_mongo'] }
    - { role: 123mwanjemike.mongodb, flags: ['configure_mongo'] }
    - { role: 123mwanjemike.mongodb, flags: ['prepare_members'] } # Destructive for existing installation!
    - { role: 123mwanjemike.mongodb, flags: ['init_replica'] }
    - { role: 123mwanjemike.mongodb, flags: ['add_shard'] }
    - { role: 123mwanjemike.mongodb, flags: ['create_database'] }
```

## Linting
```bash
yamllint -c yamllint.yaml .
ansible-lint .
```

#### References
  - [Security Hardening](https://docs.mongodb.com/manual/core/security-hardening/)
  - [Deploy Shard Cluster](https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/)
  - [Add Shards to Cluster](https://docs.mongodb.com/manual/tutorial/add-shards-to-shard-cluster)
  - [Authorization](https://docs.mongodb.com/manual/core/authorization/)
  - [Internal Auth](https://docs.mongodb.com/manual/core/security-internal-authentication/)
  - [Enforce Keyfile Access Control](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/)
  - [Deploy Replica Set w/ Keyfill Access Control](https://docs.mongodb.com/v3.2/tutorial/deploy-replica-set-with-keyfile-access-control/)