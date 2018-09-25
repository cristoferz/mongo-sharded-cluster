# MongoDB Sharded Cluster
Process for creation of a MongoDB Sharded Cluster based on Docker containers on local machine. For now on, all storage is inside container, so it is not recommended to be used on a production environment, but this can be changed by mapping volumes on instances.

Distribution

This tutorial is based on description from https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/

## Creating Docker Network

It is necessary to create a docker network for containers be able to see one another. In this scenario we will create a single network that will be used for all purposes.

This network needs its IPs to be managed by the user. So, the command for its creation is like this:

`docker network create --ip-range=172.18.0.0/24 --subnet=172.18.0.0/16 mongo-sharded-cluster`

The subnet and ip-range can be adjusted according to his needs.

## Creating ConfigSvr Replica Set

To use mongos is necessary to exist a Replica Set specifically for configuration of shards. This example creates 2 instances:

```
docker run --name mongo-cfg01 --network=mongo-sharded-cluster --ip=172.18.0.3 -d mongo:3.6 --configsvr --replSet cfg0 --bind_ip localhost,172.18.0.3

docker run --name mongo-cfg02 --network=mongo-sharded-cluster --ip=172.18.0.4 -d mongo:3.6 --configsvr --replSet cfg0 --bind_ip localhost,172.18.0.4
```

In both instances the default port is 27019, because of cluster role --configsvr. Is necessary to connect to the primary instance and initiate the replica set for the config server. To connect use this command:

`docker exec -it mongo-cfg01 mongo --port 27019`

Inside docker console initiate the replica set with this command, followed by its output:

```
> rs.initiate();
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "172.18.0.3:27019",
	"ok" : 1,
	"operationTime" : Timestamp(1537894362, 1),
	"$gleStats" : {
		"lastOpTime" : Timestamp(1537894362, 1),
		"electionId" : ObjectId("000000000000000000000000")
	},
	"$clusterTime" : {
		"clusterTime" : Timestamp(1537894362, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
cfg0:SECONDARY> 

```

After few seconds the cursor will change to cfg0:PRIMARY. Then is time to add secondary replica to the set. This step can be repeated to add multiple instances.

```
cfg0:PRIMARY> rs.add({ host: "172.18.0.4:27019", priority:0, votes: 0 });
{
	"ok" : 1,
	"operationTime" : Timestamp(1537894388, 1),
	"$gleStats" : {
		"lastOpTime" : {
			"ts" : Timestamp(1537894388, 1),
			"t" : NumberLong(1)
		},
		"electionId" : ObjectId("7fffffff0000000000000001")
	},
	"$clusterTime" : {
		"clusterTime" : Timestamp(1537894388, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
cfg0:PRIMARY>
```

The replicas is added with priority 0 and votes 0 to avoid problem on replica set initialization and then cannot assume as primary in case of failure of the actual primary. To change this behavior you can use the following sequence of commands:

```
cfg = rs.conf();
cfg.members[1].priority = 1;
cfg.members[1].votes = 1;
rs.reconfig(cfg);
```

# Creating mongos Router

With a configured Config Replica Set is possible to start the mongos router. This can be done easily by using the cristoferz/mongos docker image.

```
docker run --name mongos -p 27017:27017 --network=mongo-sharded-cluster --ip=172.18.0.2 -d cristoferz/mongos:3.6 --configdb cfg0/172.18.0.3:27019 --bind_ip localhost,172.18.0.2:27017
```

You can connect to the mongos server via this command:

`docker exec -it mongos mongo`

On this container the port 27017 is mapped to the host, so it is also possible to connect directly to the mongos via external application.

# Creating the Shards

Now will really be created the databases. We start by creating a single replica set, but many more can be created. We will create 3 instances on this case:

```
docker run --name mongo-repl01 --network=mongo-sharded-cluster --ip=172.18.0.10 -d mongo:3.6 --shardsvr --replSet repl0 --bind_ip localhost,172.18.0.10
docker run --name mongo-repl02 --network=mongo-sharded-cluster --ip=172.18.0.11 -d mongo:3.6 --shardsvr --replSet repl0 --bind_ip localhost,172.18.0.11
docker run --name mongo-repl03 --network=mongo-sharded-cluster --ip=172.18.0.12 -d mongo:3.6 --shardsvr --replSet repl0 --bind_ip localhost,172.18.0.12
```

The Shard instances will start in port 27018 by default. You can connect to the via:

`docker exec -it mongo-repl01 mongo --port 27018`

As done with the Config Replica Set, is necessary to initiate the replica set and then add the secondary replicas with priority and votes equals to 0.

```
> rs.initiate();
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "172.18.0.10:27018",
	"ok" : 1
}
repl0:SECONDARY> 
repl0:PRIMARY> rs.add({ host: "172.18.0.11:27018", priority:0, votes: 0 });
{ "ok" : 1 }
repl0:PRIMARY> rs.add({ host: "172.18.0.12:27018", priority:0, votes: 0 });
{ "ok" : 1 }
repl0:PRIMARY> 
```

And then, after all added members became secondary, adjust priority and votes:

```
cfg = rs.conf();
cfg.members[1].priority = 1;
cfg.members[1].votes = 1;
cfg.members[2].priority = 1;
cfg.members[2].votes = 1;
rs.reconfig(cfg);
```

This process can be used to create as many shards with as many replicas as necessary.

# Connecting to the Shards

The final step is to connect the shards with the mongos router. This can be done by the sh.addShard() function, connected to the mongos:

```
sh.addShard("repl0/172.18.0.10:27018");
```

Just one member of the shard needs to be specified.

# Final Considerations

This was the necessary steps for creating a MongoDB Sharded Cluster environment for testing purposes. Next steps is to enable sharding on a databases and start sharding collections. 
