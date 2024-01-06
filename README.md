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
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['install_mongo']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['configure_mongo']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['clear_logs']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['prepare_members']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['start_mongo']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['init_replica']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['create_admin']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['add_shard']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['create_database']}"
ansible-playbook playbooks/mongodb.yml -e "{'flags': ['stop_mongo']}"
```

## Flags and Variables
| Flag                 | Purpose                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| install_mongo        | Installs mongo packages                                                          |
| start_mongo          | Starts mongo database and/or server                                              |
| stop_mongo           | Stops mongo database and/or server                                               |
| configure_mongo      | Configures mongo                                                                 |
| prepare_members      | Prepares the mongodb sharded cluster members. **Deletes existing mongodb data!** |
| init_replica         | Initializes the replica set configuration                                        |
| create_admin         | Creates the admin user                                                           |
| add_shard            | Adds a shard replicaset server to the sharded cluster                            |
| create_database      | Creates a database with sharding enabled. Creates respective user if not present |
| clear_logs           | Clears all the sharded cluster logs                                              |


###### Sample playbook
```yaml
- hosts: all
  vars:
    auth_db: "" # Optional: default is admin
    # Used to initialize the replica set
    adminUser: ""
    adminPass: ""
    # Used to add a shard replicaset
    new_shard:
      name: "shard-0" # shard name
      group: "shard_0" # group name from inventories/hosts
    # Used to create a database instance on the cluster
    tgt_db: ""
    roles: ["readWrite", "userAdmin"]
    # Used to create a basic user for the database instance above
    userName: ""
    userPass: ""

  roles:
    - { role: 123mwanjemike.ansible_mongodb, flags: ["install_mongo"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["configure_mongo"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["clear_logs"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["prepare_members"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["start_mongo"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["init_replica"], when: cluster_role != 'router' }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["create_admin"], when: cluster_role != 'router' }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["add_shard"], when: cluster_role == 'router' }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["add_shard"], when: cluster_role == 'router' }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["stop_mongo"] }
```


#### References
  - [Security Hardening](https://docs.mongodb.com/manual/core/security-hardening/)
  - [Deploy Shard Cluster](https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/)
  - [Add Shards to Cluster](https://docs.mongodb.com/manual/tutorial/add-shards-to-shard-cluster)
  - [Authorization](https://docs.mongodb.com/manual/core/authorization/)
  - [Internal Auth](https://docs.mongodb.com/manual/core/security-internal-authentication/)
  - [Enforce Keyfile Access Control](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/)
  - [Deploy Replica Set w/ Keyfill Access Control](https://docs.mongodb.com/v3.2/tutorial/deploy-replica-set-with-keyfile-access-control/)