# Data Migration Strategy for HA Chef Backend Cluster

## Overview
`knife-ec-backup` is Chef's tool for migrating data between Chef server clusters of varying topologies (Standalone, Tier, old-HA, new-HA) and versions (with some limitations).   Unlike Chef's filesystem-based backup/restore tools (`pg_dumpall` or `chef-server-ctl backup|restore`), it is going to be more flexible. Filesystem-based backup/restore tools can ONLY be used to backup/restore LIKE versions and LIKE topologies of Chef Server. `knife-ec-backup` on the other hand, creates a highly portable object-based backup of the entire Chef server including all Orgs, Users, Cookbooks, Databags, etc.

Because Chef HA does not have an in-place migration tool, the expectation is that you will build and validate a new Chef HA Cluster and then migrate your data to it.

The steps to building a new Chef HA Cluster are as follows:
1. Build a new Chef HA Cluster (bonus points for using [automation](https://github.com/ncerny/chef_stack) to reproducibly build your cluster)
2. Validate your new Chef HA Cluster  (we cannot stress enough the importance of this)
    * Test each new frontend using `chef-server-ctl test`
    * Test both hard and soft failovers of the backend systems
    * Load test your new cluster using [chef-load](https://github.com/jeremiahsnapp/chef-load)
3. Perform a full backup of your current production Chef Server cluster.  Note:
    * `knife-ec-backup` can generate considerable load on your cluster, particularly when increasing parallelism.
    * It is recommend that you upgrade to the latest versions of Chef Server if at all possible.   Using the latest version of `knife-ec-backup` is a requirement.
    * If you begin to experience 500 errors on your existing Chef Server during a full backup, attempt these during off-peak hours if possible. 
    * Performance tuning of your Chef server may be required.  For more information see:
        * [Monitoring and Tuning your Chef Server](https://www.slideshare.net/AndrewDuFour1/monitoring-and-tuning-your-chef-server-chef-conf-talk)
        * [Tuning the Chef Server for Scale](http://irvingpop.github.io/blog/2015/04/20/tuning-the-chef-server-for-scale/)
        * [Understanding the Chef Server](https://www.youtube.com/watch?v=22GtVMHJDsI)
    * An advanced strategy may be to temporarily add a dedicated Frontend to existing Tier/HA topologies in order to reduce loading on the remaining frontends
4. Perform a full restore to the new cluster
    * The performance monitoring and tuning advice from the previous step will help achieve higher levels of parallelism
    * There may be errors encountered during restore - for example expired user-org invitations that point to deleted users.  It is recommended that you fix as many of these as possible on the source server.   If that's not possible, it's possible to fix the errors directly in the JSON filess as an intermediate "data cleanup" stage before restoring.
5. Validate the target cluster
    * Retest the cluster both `chef-server-ctl test` as well as re-pointing a number of non-critical nodes at the new cluster
6. Perform nightly incremental backups/restores
    * `knife-ec-backup` can operate in a pseudo-incremental mode as long as you keep the backup directory intact.  Continue to run backups/restores and you'll notice they complete much faster than the original
    * Note the time it takes for an incremental backup/restore to complete - this should provide you with the clearest guidance for how long of a downtime/maintenance window to schedule 
    * An advanced strategy is to migrate one org or batches of orgs at a time.  In this case you'll need to:
        * Use the chef-client cookbook or similar strategy to update the client.rb file on every node to point at the new cluster
        * Filter already-migrated orgs from the backup/restore once migration is complete

### Backup / Restore Process

Install the [ChefDK](https://downloads.chef.io/chefdk)

Install latest knife-ec-backup gem from rubygems.org
```bash
chef gem install knife-ec-backup
```

Create a local backups working directory
```bash
mkdir -p chef_backups/conf
```

Create a local `knife.rb` for the SOURCE and DESTINATION Chef Servers from which we'll export then import the data
```bash
cd chef_backups/conf
cat <<EOF> knife_src_server.rb
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
chef_server_url          "https://my.chef.com/organizations/brewinc"
ssl_verify_mode          :verify_peer
EOF
cat <<EOF> knife_dst_server.rb
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
chef_server_url          "https://my.chef-ha.com/organizations/brewinc"
ssl_verify_mode          :verify_peer
EOF
```

Fetch the ssl certs
```bash
knife ssl -c knife_src_server.rb fetch
knife ssl -c knife_dst_server.rb fetch
```

Copy the `/etc/opscode/webui_priv.pem` file from both the SOURCE and DESTINATION Chef Servers locally into `chef_backups/conf` giving them unique names.

### Infrastructure
Since `knife-ec-backup` uses the Chef Server API entirely it is more agnostic with regards to the Chef Server version and topology deployed.    For that reason, `knife-ec-backup` is a perfect candidate to assist with migrating from a standalone or tiered toplogy to an HA backended cluster.

**Note:** As a pre-requisite to a migration, you should upgrade both the source and destination Chef servers to the most recent version available, and the version shold match across all Chef servers.

### Backup from SOURCE

By default `knife-ec-backup` will use a concurrency of 10.  We've found in testing that a concurrency of 50 seems to be a sweet spot and yields significant improvement.  Any higher than 50 and you will likely reach a point of diminishing returns.  Also, it is important to keep in mind that concurrency can cause a spike in the load on your Chef Server.  Thus, keep a close watch on your Chef Server stats while running a backup and terminate the backup if returning excessive HTTP 500s to chef client agents.

**IMPORTANT:** To add additional concurrency threads, append `--concurrency 50` to the following command:

```bash
cd ../..
knife ec -c chef_backups/conf/knife_src_server.rb backup chef_backups --webui-key chef_backups/conf/webui_priv_src.pem
```

The command above will download all data from the source Chef Server and store objects as individual json files beneath `chef_backups`.  It is safe to re-run the backup multiple times over the existing `chef_backups` directory.  On subsequent runs, `knife-ec-backup` will do a differential backup of the `/cookbooks` objects.

**Note:** Because the `backup` operation can be run multiple times and only NEW cookbooks added to the source will be transferred down, one good strategy may be to run repeated backups ahead of the migration day if the initial backup takes a prohibitively long time. Running several small backups ahead of time may be better than running one BIG one. Another supporting strategy might be performing backups during low-peak times, or adding frontend capacity during both backups (db connections) and restores (frontend CPU bound).

### Restore to new DESTINATION

Again, for best results, the recommended strategy is to utilize a new destination cluster without any pre-existing data.

```bash
knife ec -c chef_backups/conf/knife_dst_server.rb restore chef_backups --webui-key chef_backups/conf/webui_priv_dst.pem
```

**IMPORTANT NOTE:** Unlike a backup operation, the default concurrency of 10 seems to yield the best results on a restore.  Bumping up the concurrency any further will not produce any gains.

If `knife-ec-backup` receives a 500 response while restoring, it will retry the operation up to 5 times with ever increasing delay between retries.  After the 5th failed attempt, the job will abort.  As with a backup, it is safe to re-run the restore multiple times to get a complete run.

### Testing Results
During our testing we set up an environment with the following, utilizing an implementation of the resource cookbook [chef_stack](https://github.com/ncerny/chef_stack):
- Standalone Chef Server (source) with 1,600 cookbook versions and 10,000 nodes
- Backended HA Cluster (destination) consisting of 3 backends and 2 frontends
- The ec-backup of this configuration resulted in 1GB of total data stored on the workstation.

Type | Concurrency | Optimization Used? | Duration (mins)
--- | --- | --- | ---
backup | 10 (default) | No | 14
backup | 50 | No | 5
restore | 10 (default) | No | ~120
restore | 50 | No | > 120
restore | 50 | Yes | 15

### Discoveries
1. Bumping the concurrency setting significantly improves backup times while having a detrimental effect on restores.
2. Cookbook backup and restores are incremental and only transferred if the src/dst is different.
3. As of version 2.0.7 of `knife-ec-backup`, Node Objects are the least optimized item on restores, significantly impacting the overall time.

### Optimization
The reason for #3 above is primarily because the Chef Server API does not have a low cost method for interrogating if a node exists or not. The API exposes `GET /organizations/NAME/nodes/NAME` which will return the entire node object to the requestor.  There is also `GET /organizations/NAME/nodes` which returns a hash of all the node names. `knife-ec-backup` uses the latter to interrogate if the node exists or not because the API endpoint for updating a node object is a different endpoint than the one for creating a node new node object and it needs to know which operation to use on the restore: either create or update.

To create a new node, you use `POST /organizations/NAME/nodes` posting the node object.  To update a node, you use `PUT /organizations/NAME/nodes/NAME`.  Therefore, on EACH node object restore, `knife-ec-backup` issues a `GET /organizations/NAME/nodes` and loops through the hash to determine if the node name exists or not.  This is not terrible unless the node count increases above say 1,000 then the hash being returned on every node request combined with the operation of looping through to determine if the node key exists becomes prohibitively expensive.

All the above to say: the greater the node count the more inefficient the restore operation becomes and it actually slows down as it makes its way through restoring the nodes to the target.

To workaround this short term issue, until `knife-ec-backup` can be corrected or the Chef Server API improved, we've created a ruby script that uses a different approach for node restores.

The script below assumes no nodes exist and therefore does not request the node list, but just does a `POST /organizations/NAME/nodes` to create the node, if it receives a `HTTP 409 Conflict` response, the node exists so it follows up with a `PUT /organizations/NAME/nodes/NAME` to update the existing node. It uses Threading, allowing you to set the number of worker threads but defaults to 50.

As shown in the table above, this improved the overall restore operation of our data set from >1.5 hours to 15 minutes.

In order to use this optimization on a restore, rename the `nodes` directory to something that `knife-ec-backup` will ignore, like `mynodes`.  Run the `knife-ec-backup` restore operation as described above then follow up by running the script below:

```bash
~/Devel/ChefProject$ ./restore_nodes.rb -h
Usage: ./restore_nodes.rb [options]
    -c, --config PATH                path to knife/client config file
    -n, --nodes PATH                 path to nodes directory
    -h, --help                       Show this message
~/Devel/ChefProject$ ./restore_nodes.rb -c ~/.chef/knife.rb -n path/to/mynodes
```

**Note**: You will have to supply a valid `knife.rb` file with an admin's credentials (potentially re-use the pivotal key from the steps above), for each org in the backup.

Since this script needs to be run per-org, if you have multiple orgs in the backup, then consider serializing the restore with: `knife ec restore --only-org ORG ..`
for each org, followed by running `restore_nodes` on that org.  The entire process would benefit from being scripted.

```ruby
#!/usr/bin/env ruby

require 'chef'
require 'thread'
require 'optparse'

ARGV << '-h' if ARGV.length != 4

options = {}
OptionParser.new do |opts|
  opts.banner = "Usage: #{$PROGRAM_NAME} [options]"
  opts.on('-c', '--config PATH', 'path to knife/client config file') { |v| options[:config] = v }
  opts.on('-n', '--nodes PATH', 'path to nodes directory') { |v| options[:nodes] = v }
  opts.on_tail('-h', '--help', 'Show this message') do
    puts opts
    exit
  end
end.parse!

Chef::Config.from_file(options[:config])
REST = Chef::ServerAPI.new(Chef::Config[:chef_server_url])

queue = Queue.new
Dir.glob("#{options[:nodes]}/*.json").each { |n| queue.push n }

def sendnode(node)
  printf("Creating %s\n", node['name'])
  REST.post_rest('/nodes', node)
rescue Net::HTTPServerException => e
  case e.message
  when /409/
    REST.put_rest("/nodes/#{node['name']}", node)
  end
  printf("  #{node['name']}: #{e.message}\n")
end

workers = (0...5).map do
  Thread.new do
    begin
      while (node_file = queue.pop(true))
        sendnode(JSON.parse(File.read(node_file)))
      end
    rescue ThreadError
      true
    end
  end
end
workers.map(&:join)
```

The above code can be found in this [gist](https://gist.github.com/jeremymv2/85775008c8b4d6d49e2c4f8ce2da4825)

### Potential Improvements to `knife-ec-backup` and/or `chef-client`, `chef-server`.
- Modifying the behavior of `knife-ec-backup` to incorporate the strategy in the above workaround: Just sending a POST and if a 409 is received, send a PUT.
- Adding the capability of uploading a tarball for both cookbooks and nodes and just letting `chef-server` handle it
- Modifying the `chef-server` API so that there is a way to query if a node exists without having to transfer the entire node object.  This is of lesser value to the use-case scenario of restoring to a blank destination, but just seems like a good idea in general.
- A variation of the above is the ability to modify `chef-client`'s `Chef::ServerAPI` to be able to request HEAD on `/nodes/name` so that the full node object isn't returned in the request and look for a 200 or 404 response.
- Additional information in the `knife-ec-backup` [README.md](https://github.com/chef/knife-ec-backup/blob/master/README.md)
