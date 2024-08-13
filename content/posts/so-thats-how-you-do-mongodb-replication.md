+++
title = "So That's How You Do MongoDB Replication!"
date = 2024-08-13T15:05:40+07:00
images = ["/so-thats-how-you-do-mongodb-replication.webp"]
+++

When I first joined [Indorelawan](https://www.indorelawan.org) as a fresh-faced graduate, I had my fair share of battles with MongoDB replication. We had two servers: one for the primary and one for the secondary. Simple, right? Well, not so much. Every time the secondary server decided to take a nap, the primary would throw in the towel too. It was a complete disaster, and I’ll admit--I got so frustrated that I just ripped the replication feature out entirely (not my proudest moment, but hey, desperate times).

Fast forward to 2024, and I'm still helping Indorelawan out at [Hyperjump](https://hyperjump.tech). This time around, I needed to set up an ELT system using Airbyte, and guess what? MongoDB replication is back on the table. But I’m older, wiser, and determined to get it right this time. So I did.

## Here comes the tutorial

After scouring countless sources and banging my head against the wall a few times, I finally got it: you need three servers, not just two. The first two servers are for the primary and secondary, but the third one? That’s for the arbiter.

According to the [MongoDB docs](https://www.mongodb.com/docs/manual/core/replica-set-arbiter/):

> "In some circumstances (such as when you have a primary and a secondary, but cost constraints prohibit adding another secondary), you may choose to add an arbiter to your replica set. An arbiter participates in elections for primary but an arbiter does not have a copy of the data set and cannot become a primary."

In plain English, if the primary server kicks the bucket, the arbiter steps in and says, “Hey, secondary, you’re in charge now!” The arbiter doesn’t store any data or do much heavy lifting, but it has the final say in who gets to be primary.

Now that we've got our three servers ready to roll, it's time for the execution. Before you proceed, don't forget to install MongoDB on each server. Please follow the official [MongoDB installation guide](https://docs.mongodb.com) for your specific operating system.

### Step 1: Modify the /etc/hosts file

The first thing you need to do is add these IP addresses to each server's /etc/hosts file. This step is crucial because it ensures that all three servers know how to find each other. So, on each server, you'll want to edit the /etc/hosts file and include the following lines:

```
192.168.0.2 mongo-db1
192.168.0.3 mongo-db2
192.168.0.4 mongo-arbiter
```

Remember, this setup needs to be the same on all three servers. And of course, swap out these IP addresses with the actual ones you're using in your environment.

### Step 2: Modify each server's /etc/hostname

Next up, let's make sure each server can be identified by a name rather than just an IP address. This makes things a lot more manageable and easier to remember. On each server, you'll want to rename the hostname accordingly:

On the first server (192.168.0.2):

```bash
$ sudo vim /etc/hostname
```

Change the content to:

```
mongo-db1
```

On the second server (192.168.0.3):

```bash
$ sudo vim /etc/hostname
```

Change the content to:

```
mongo-db2
```

On the third server (192.168.0.4):

```bash
$ sudo vim /etc/hostname
```

Change the content to:

```
mongo-arbiter
```

This way, each server has a friendly name, making it easy to identify who's who in your MongoDB replica set.

### Step 3: Generate a key file

To allow the servers to communicate securely without needing an external passcode, we'll use a key file. This key will authenticate the servers with each other.

Here's how you can generate the key on the first server (mongo-db1):

```bash
# Create the directory for the keys
mkdir -p /etc/mongodb/keys/

# Generate the key file
openssl rand -base64 756 > /etc/mongodb/keys/mongo-key

# Set the correct permissions to keep the key secure
chmod 400 /etc/mongodb/keys/mongo-key

# Ensure the MongoDB user owns the key
chown -R mongodb:mongodb /etc/mongodb
```

Once the key is generated on mongo-db1, you'll need to copy this key file to the same location on all other servers (mongo-db2 and mongo-arbiter):

```bash
# Example for copying the key to another node
scp /etc/mongodb/keys/mongo-key user@mongo-db2:/etc/mongodb/keys/
scp /etc/mongodb/keys/mongo-key user@mongo-arbiter:/etc/mongodb/keys/
```

This key ensures that all servers can securely communicate with each other.

### Step 4: Configure the replica set

Now that you've configured everything on the servers, there's one final step: adding the IP addresses of the servers and specifying the replication set name. You'll do this by modifying the MongoDB configuration file located at /etc/mongo.conf.

On the first server (mongo-db1), open the configuration file and update it as follows:

```
# network interfaces
net:
  port: 27017
  bindIp: 192.168.0.2

# security settings
security:
  authorization: enabled
  keyFile: /etc/mongodb/keys/mongo-key

# replication settings
replication:
  replSetName: "replicaset-01"
```

Repeat this process on the other two servers, ensuring that each server's configuration reflects its specific IP address. Once you've made these changes on all the servers, restart the MongoDB services to apply the new configurations:

```bash
sudo systemctl restart mongod
```

### Step 5: Initiate the replica set

With everything configured, it's time to kick off the replication process. Start by logging into the primary server, 192.168.0.2:

```bash
$ mongosh <mongodb-connection-string>
```

Once you're in the MongoDB shell, initiate the replica set with the following command:

```bash
> rs.initiate();
```

This command tells MongoDB to start the replication process, setting up the primary and secondary servers along with the arbiter. Once the replication REPL is started, it’s time to add nodes to the replication set by the following command:

```bash
> rs.add(“mongo-db2:27017”);
> rs.addArb(“mongo-arbiter:27017”);
```

Once you add the servers, you will see the output as {‘ok’:1}, which indicates a successful addition of servers in the replica set.

### Step 6: Verify the replica set

To check the status of your replication set and ensure everything is running smoothly, you can use the following command in the MongoDB shell:

```bash
> rs.status()
```

This command will give you a detailed overview of your replica set, including the state of each node. Here’s a sample output you might see:

```
{
  set: 'replicaset-01',
  date: ISODate("2024-08-13T08:35:54.411Z"),
  myState: 1,
  ...(omitted for brevity),
  members: [
    {
      _id: 0,
      name: '192.168.0.2:27017',
      health: 1,
      state: 1,
      stateStr: 'PRIMARY',
      uptime: 2867565,
      optime: { ts: Timestamp({ t: 1723538152, i: 32 }), t: Long("40") },
      optimeDate: ISODate("2024-08-13T08:35:52.000Z"),
      self: true,
    },
    {
      _id: 1,
      name: '192.168.0.3:27017',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 2867554,
      optime: { ts: Timestamp({ t: 1723538152, i: 32 }), t: Long("40") },
      optimeDurable: { ts: Timestamp({ t: 1723538152, i: 32 }), t: Long("40") },
      optimeDate: ISODate("2024-08-13T08:35:52.000Z"),
    },
    {
      _id: 2,
      name: '192.168.0.4:27017',
      health: 1,
      state: 7,
      stateStr: 'ARBITER',
      uptime: 2867407,
      lastHeartbeat: ISODate("2024-08-13T08:35:54.334Z"),
      lastHeartbeatRecv: ISODate("2024-08-13T08:35:54.136Z"),
    }
  ],
  ok: 1,
  ...(omitted for brevity),
}
```

In this output:

- "stateStr": "PRIMARY" indicates that the server is the primary server.
- "stateStr": "SECONDARY" shows that the server is acting as a secondary server.
- "stateStr": "ARBITER" means the server is functioning as the arbiter.

If everything looks good, your MongoDB replica set is officially up and running!

## Wrapping It Up

And there you have it! Your MongoDB replica set is now up and running. If the primary server decides to take an unscheduled vacation, your secondary server and arbiter will keep things from turning into a full-blown data disaster.

It’s funny to look back and remember how MongoDB replication once felt like a nightmare. But with a bit of persistence and a lot of trial and error, I've turned that challenge into a triumph.

So, next time you’re tempted to disable replication, just remember: three servers are better than one, and they make sure your data doesn’t pull a Houdini act.

And remember, if all else fails, there’s always coffee. Lots of coffee.
