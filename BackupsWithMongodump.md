# Backups with ```mongodump``` and ``mongorestore``

This exercise will work through an example of taking a backup from
a MongoDB replica set and then restoring the backup.

There are multiple strategies for doing backups with MongoDB, here we
are going through a basic example of using MongoDB tools to do a backup
on a replica set. This is only an example; please use at your own risk 
and please test your procedures thouroughly before running this on 
any production systems.

The exercise assumes you have a running replica set.

#High Level Steps#
1. Chose a secondary to run the backup on
2. Pause the secondary
3. Run the ```mongodump``` tool
4. Un-pause the secondary
5. Compress the backup
6. Restore the backup

#Setup#

To start, let's seed some sample data into our replica set.
Open a text editor, copy/paste the following javascript, and
then save the file as ```sampleDataLoader.js```.

```
db = db.getSiblingDB("test")
for (var i=0;i<1000;i++) { db.data.insert({"value" : i, "ts" : new Date()}); }
print("db.data.count()="+db.data.count());
```
Connect to your primary node and run the script:

```
$mongo <connection_info> sampleDataLoader.js
```

You should verify and validate that you have 1000 documents in the ```data```
collection.

#Create the backup#
Now we can take a backup. You will need 2 command prompts to complete
this part.

In your first prompt, open a ```mongo``` shell to the secondary you wish to take
a backup on and run the following:

```
SECONDARY:replSet>print(new Date()); db.fsyncLock()
Tue Jun 07 2016 21:41:33 GMT-0400 (EDT)
{
	"info" : "now locked against writes, use db.fsyncUnlock() to unlock",
	"seeAlso" : "http://dochub.mongodb.org/core/fsynccommand",
	"ok" : 1
}
```

**keep this shell open** and **do not close this shell session until we're
done with the backup!**
Note, here you are actually running 2 things. Printing out the current date/time
and running the lock command. We print out the date information so there is a
record of when the data was frozed; we'll use this later to archive off the data
and label it appropriately.

Now, in your second comment prompt we'll use the ```mongodump``` tool
to dump out BSON files with all the data in all the collections:

```
$mongodump <connection_info>
$ls -l dump
```

By default ```mongodump``` will write the backup files into a directory
called ```dump```. You can control the output directory, run 
```mongodump --help``` to see all the options available.

Now we can unlock the secondary, so go back to the shell session
and issue the following:

```
SECONDARY:replSet>db.fsyncUnlock()
{ "info" : "unlock completed", "ok" : 1 }
```

At this point your replica set is back to normal operational status.

#Archive the backup#
Now you have a directory containing your backup. You'll probably want to 
save this data off somewhere safe, it's a backup afterall. On Linux based
systems, the following command will compress the backup data into
one file:

```
$tar czvf mongo-backup-2016-06-07-21-41-33-GMT-0400.tar.gz dump
```

*note your datetime will be different so replace accordingly*.

#Add some more data#
Before we practice a restore, let's write some more data to the replica set.
This way once we've restored, we can prove to ourselves that the data has 
reverted to the perivous state when we make the backup.

Run the ```sampleDataLoader.js``` script again against your primary node.

#Restore the backup#

Since this is a DR (disaster recovery) situation, our replica set is going to 
have to take some down-time in order to restore the backup. There is not a way
to restore a backup without downtime (restoring backups does not work like 
"rolling-maintenance"").

First, make sure the applications connecting to your MongoDB are shutdown or paused if 
possible. If not, once we bring down the replica set your application will start to 
generate lots of errors.

Before we start shutting down the replica set, run the  ```ps -ef | grep mongod``` command 
and take note of how the ```mongod```'s are running. We'll need this information later.

Gracefully shut down all the nodes in your replica set, starting with the current
secondaries:

```
SECONDARY:replSet>use admin
SECONDARY:replSet>db.shutdownServer()
```

Finally connect to the original primary (which by this point will be in ```SECONDARY``` mode 
and read only) and shut it down just like above.

The restore strategy can follow a couple of paths here, but generally the idea will be to
delete all the data files in each node and then use the ```mongorestore``` tool to load the data
in. The simplest approach is to just let each secondary resync the data - this can be 
prohibitive however with large dataset since it takes time. 

##Simple restore with re-syncs##

There are options here as well. We could either blow away all the data files on each node and restore.
Or we could manually drop all the databases and then restore. If you blow away all the data files, then 
you'll also wipe out the replica set configuration, which may not be ideal should you have a complicated
configuration. So, for this example, we take the later option. **But** you must make sure no applications
will connect to the replica set. One way to prevent this is for sure, is to reconfigure your network
DNS settings to make the MongoDB nodes unavailable. Or you could edit the configuration files for your 
```mongod```'s and manipulate the [bindIp](https://docs.mongodb.com/manual/reference/configuration-options/#net.bindIp)
parameter to only listen on 127.0.0.1 which will in 
effect only allow connections from the same server the ```mongod``` is running. This exercise
will chose this later option as well.

Our plan here will be to chose one node to load the data into and then let it replicate.
Pick one node and reconfigure it to run in standalone mode and only list on the localhost
interface.

If you run your ```mongod```'s with a config file, edit the chosen node's config file adding the
net.bindIp and commenting out the replication settings (comment out all replication settings):

```
net:
   bindIp: 127.0.0.1
...
#replication:
#   replSetName: replSet
```

If you're running your replica set with command line parameters, then add the following to your
```mongod``` command:

```
$mongod <other options> --bind_ip 127.0.0.1
```

and **remove** the ``--replSet`` arugment. 


Now restart the node we chose to load the data, and connect a shell. Note that since
we changed the configuration, you'll only be able to connect a ```mongo``` shell from session on
the server. Note - you should see a warning message about the instance running without the ```--replSet```
aregument but replication configuration existing.

From your ```mongo``` shell:

```
>show dbs
```

We want to drop all these databases **except** the ```admin``` and ```local``` database. These are 
reserved system databases.

For each database in your instance run: ```db.getSiblingDB(<DB_NAME>).dropDatabase()```. Where you
should replace ```DB_NAME``` with the real names of your databases. For example:

```
PRIMARY:replSet>db.getSiblingDB("test").dropDatabase()
{ "dropped" : "test", "ok" : 1 }
```

We can now restore.

Let's take our archive created before and uncompress it (you'll probably want
to uncompress in some temp directory). Assuming our archive is in a directory called 
``` /archives/mongodb ```:


```
$cd /temp
$mkdir mongo-backup-2016-06-07-21-41-33-GMT-0400
$cd mongo-backup-2016-06-07-21-41-33-GMT-0400
$tar xzvf /archives/mongodb/mongo-backup-2016-06-07-21-41-33-GMT-0400.tar.gz 
```

Now run the ```mongorestore``` command:

```
$pwd
/temp
$mongorestore <connection_info> mongo-backup-2016-06-07-21-41-33-GMT-0400/dump
```

This will load all the data into the temporary standalong node.

Getting back to a fully functional state is straighforward now.
Stop the standalone node you just loaded the data.

For each other node in the replica set, figure out the ```--dbpath``` and delete all the files in 
this directory. For example:

```
$cd /data/db
$rf -Rf *
```

Back on node we loaded data into, revert the config changes: remove the ```bindIp``` 
and uncomment out the replication settings.
Restart this node we loaded the data into & ensure it starts correctly and then restart all your nodes.
Each secondary should now begin to resync all it's data.

##More complex, restore with pre-seeding##

**SECTION NOT COMPLETE YET**

https://docs.mongodb.com/manual/tutorial/resync-replica-set-member/#sync-by-copying-data-files-from-another-member

Now, restart

