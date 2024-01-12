# ansible-mongodb

Purpose: Create a MongoDB sharded cluster from scratch.

#### Getting started
1. Start by adding the following to your `requirements.yml` file:
    ```yaml
    -   name: 123mwanjemike.ansible_mongodb
        src: https://github.com/123MwanjeMike/ansible-mongodb
        version: v1.0.4
    ```
2. Install the role by running the ansible-galaxy install command:
    ```bash
    ansible-galaxy install -r requirements.yml
    ```

#### Required ansible host/group variables

For each host/group, the `cluster_role` is either of *replicaSet*, *router*, *config* or *shard*.

The `replica_set` is also required for host groups that are in a replica set and is a dictionary with the following keys:
- name: The name of the replica set
- group: The name of the host group in the inventory that the replica set belongs to

Below is an example:
```yaml
cluster_role: "config"
replica_set:
    name: "cfgsvr"
    group: "config_servers"
```

You may also checkout the values in the [vars/main.yml](vars/main.yaml) file and change them according to needs.

#### Sample inventory
```yaml
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3
  children:            
    mongo_cluster:
      children:
        mongos_routers:
          hosts:
            mongos-router-0.europe-north1-b.c.oceanic-muse-408212.internal:
          vars:
            cluster_role: "router"
        config_servers:
          hosts:
            mongod-cfgsvr-0.europe-north1-b.c.oceanic-muse-408212.internal:
            mongod-cfgsvr-1.europe-north1-c.c.oceanic-muse-408212.internal:
            mongod-cfgsvr-2.europe-north1-a.c.oceanic-muse-408212.internal:
          vars:
            cluster_role: "config"
            replica_set:
              name: "cfgsvr"
              group: "config_servers"
        shard_0:
          hosts:
            mongod-shard-0-0.europe-north1-b.c.oceanic-muse-408212.internal:
            mongod-shard-0-1.europe-north1-c.c.oceanic-muse-408212.internal:
            mongod-shard-0-2.europe-north1-a.c.oceanic-muse-408212.internal:
          vars:
            cluster_role: "shard"
            replica_set:
              name: "shard-0"
              group: "shard_0"
        plain_replica_set:
          hosts:
            mongod-rs0-0.europe-north1-b.c.oceanic-muse-408212.internal:
            mongod-rs0-1.europe-north1-c.c.oceanic-muse-408212.internal:
            mongod-rs0-0-2.europe-north1-a.c.oceanic-muse-408212.internal:
          vars:
            cluster_role: "replicaSet"
            replica_set:
              name: "rs0"
              group: "plain_replica_set"
```

**NOTE**: It is important that the hostnames are the FQDNs of the hosts as shown in the example above.

#### Flags and Variables
| Flag            | Variables                         | Description                                                                      |
| --------------- | --------------------------------- |--------------------------------------------------------------------------------- |
| install_mongo   | none                              | Installs mongo packages                                                          |
| start_mongo     | none                              | Starts mongo database and/or server                                              |
| stop_mongo      | none                              | Stops mongo database and/or server                                               |
| configure_mongo | cfg_server                        | Configures mongo                                                                 |
| prepare_members | keyfile_src                       | Prepares the mongodb sharded cluster members. **Deletes existing mongodb data!** |
| init_replica    | none                              | Creates a mongodb replica set                                                    |
| create_admin    | adminUser, adminPass              | Creates the admin user                                                           |
| add_shard       | new_shard                         | Adds a shard replicaset server to the sharded cluster                            |
| create_database | tgt_db, roles, userName, userPass | Creates a database with sharding enabled. Creates respective user if not present |
| clear_logs      | none                              | Clears all the sharded cluster logs                                              |


##### Sample playbook
```yaml
- hosts: all
  vars:
    # Used to create admin user
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
    # Used to prepare the members of the sharded cluster
    keyfile_src: "./keyfile" # Path to the keyfile on the ansible controller
    # Used to configure the cluster members
    cfg_server:
      name: "cfgsvr"
      group: "config_servers"

  # Generate a keyfile
  pre_tasks:
    - name: Generate random string with OpenSSL on ansible controller
      shell: openssl rand -base64 756 > keyfile
      delegate_to: localhost
      args:
        creates: keyfile

  roles:
    - { role: 123mwanjemike.ansible_mongodb, flags: ["install_mongo"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["configure_mongo"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["clear_logs"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["prepare_members"] }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["start_mongo"] }
    - {
        role: 123mwanjemike.ansible_mongodb,
        flags: ["init_replica"],
        when: cluster_role != 'router',
      }
    - {
        role: 123mwanjemike.ansible_mongodb,
        flags: ["create_admin"],
        when: cluster_role != 'router',
        delegate_to: "{{ groups[replica_set.group] | first }}",
      }
    - {
        role: 123mwanjemike.ansible_mongodb,
        flags: ["add_shard"],
        when: cluster_role == 'router',
        run_once: true,
      }
    - {
        role: 123mwanjemike.ansible_mongodb,
        flags: ["create_database"],
        when: cluster_role == 'router',
        run_once: true,
      }
    - { role: 123mwanjemike.ansible_mongodb, flags: ["stop_mongo"] }
```

###### Notes:
1. The setup is best suited for implementation behind a VPC and thus implements authentication using a keyfile. If you need to use x509 certificates, you can use the [x509 security](https://www.mongodb.com/docs/manual/core/security-x.509/) documentation to guide you on how to implement it.
2. This ansible role was originally templated from [this role](https://galaxy.ansible.com/ui/standalone/roles/isaackehle/ansible_mongodb/). This particular one is however intended to be light weight and easy to use and thus users that may need a more sophisticated setup can still checkout the original role.
