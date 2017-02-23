# Data Migration Strategy for HA Chef Backend Cluster
This document describes some points areound migrating data from a non-HA Chef Server to Chef HA Backend cluster.

This assumes a newly installed Chef HA backend cluster is available and not yet populated with data.


#### Create an Organization and Backup admin user on one of the Front End nodes of the new HA cluster

```
sudo chef-server-ctl user-create brewadmin Bob Backup backup@example.com securepassword --filename backupkey.pem

sudo chef-server-ctl org-create brewinc "Brew, Inc." --association_user brewadmin --filename brewinc-validator.pem
```

#### Set up your work environment on a dedicated workstation

Install the ChefDK (https://downloads.chef.io/chefdk)

Install latest knife-ec-backup gem from rubygems.org
```
chef gem install knife-ec-backup
```

Create a local backups working directory
```
mkdir -p chef_backups/conf
```

Create a local `knife.rb` for the SOURCE and DESTINATION Chef Servers from which we'll export then import the data
```
cd chef_backups/conf

cat <<EOF> knife_src_server.rb
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
chef_server_url          "https://fe1.mychef.com/organizations/brewinc"
ssl_verify_mode          :verify_peer
EOF

cat <<EOF> knife_dst_server.rb
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
chef_server_url          "https://fe1.chef-ha.com/organizations/brewinc"
ssl_verify_mode          :verify_peer
EOF
```

Fetch the ssl certs
```
knife ssl -c knife_src_server.rb fetch
knife ssl -c knife_dst_server.rb fetch
```

Copy the `/etc/opscode/webui_priv.pem` files from both the SOURCE and DESTINATION Chef Servers locally into `chef_backups/conf` giving them unique names.

Backup from SOURCE
By default `knife-ec-backup` will use a concurrency of 10.  We've found in testing that a concurrency of 50 seems to be a swee spot.  Any higher than that and you will likely will get diminishing returns.  Also, it is important to keep in mind that this will spike the load on your Chef Server - keep a close watch on your Chef Server stats while backing up.

To add additional concurrency threads, append this to the following command: `--concurrency 50`

```
cd ../..
knife ec -c chef_backups/conf/knife_src_server.rb backup chef_backups --webui-key chef_backups/conf/webui_priv_src.pem
```

The command above will download all data from the source Chef Server and store them as individual json file objects beneath `chef_backups`.  It is safe to re-run the backup over the existing `chef_backups` directory.  `knife-ec-backup` will do a differential backup of the `/cookbooks` objects, which potentially consume the longest amount of time to transfer.

Restore to DESTINATION

Import the data into your destination HA Cluster.  For best results, the recommended strategy is to utilize a new destination cluster, without pre-existing data.

```
knife ec -c chef_backups/conf/knife_dst_server.rb restore chef_backups --webui-key chef_backups/conf/webui_priv_dst.pem
```

If `knife-ec-backup` receives a 500 response, it will make multiple attempts, however sometimes the job will abort.  In that scenario and in general, it is safe to re-run the restore multiple times.

### Test Results
During out testing we replicated an environment with the following:
- Standalone Chef Server (source) with 1,600 cookbooks and 10,000 nodes
- Backended HA Cluster (3 backends, 1 frontend)

```
knife ec -c chef-backup-testing/conf/knife_src_server.rb backup  --webui-key   293.39s user 35.72s system 36% cpu 14:58.67 total
^ default concurrency, 10k nodes, 1,600 cookbooks

knife ec -c chef-backup-testing/conf/knife_src_server.rb backup  --webui-key   280.34s user 31.76s system 97% cpu 5:18.95 total
^ concurrency 50, 10k nodes, 1,600 cookbooks

restore with default concurrency failed once on cookbooks, and projected to 2 hours

no significant improvent was observed on a restore with bumping the concurrency any more than the default 10.

chef exec ruby node_upload.rb  213.04s user 12.22s system 48% cpu 7:46.88 total
custom ruby script
```

```
require 'chef'
require 'thread'

Chef::Config.from_file('/Users/nolan/chef-backup-testing/conf/knife_dst_server.rb')
REST = Chef::ServerAPI.new(Chef::Config[:chef_server_url])

queue = Queue.new
Dir.glob('/Users/nolan/chef-backup-testing/organizations/jnj/nodes/*.json').each { |n| queue.push n }

def sendnode(node)
  printf("Creating %s\n", node['name'])
  REST.post_rest('/nodes', node)
rescue Net::HTTPServerException => e
  REST.put_rest("/nodes/#{node['name']}", node) if e.message == '409 "Conflict"'
end

workers = (0...50).map do
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

### Potential Improvements
- Modifying the behavior of `knife-ec-backup` to encoropate the strategy in the above workaround: Just sending a POST and if 409, send a PUT.
- Adding the capability of uploading a tarball for both cookbooks and nodes and just letting the Chef Server handle it
- Modifying the Chef Server API so that there is a way to query if a node exists without having to transfer the entire node object.  This is of lesser value to the use-case scenario of restoring to a blank destination - the gratuitous POST is likely the best option.
- A variation of the above would be to modify the GET request on nodes to just request HEAD so that the full node object isn't returned in the request.
