# Data Migration Strategy for HA Chef Backend Cluster
This document describes the process of migrating data from a Chef Server to a Chef HA Backend Cluster.
For best results ensure that the Chef HA backend cluster has not yet populated with data.

### Set up your work environment on a dedicated workstation with adequate disk space

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
```
knife ssl -c knife_src_server.rb fetch
knife ssl -c knife_dst_server.rb fetch
```

Copy the `/etc/opscode/webui_priv.pem` files from both the SOURCE and DESTINATION Chef Servers locally into `chef_backups/conf` giving them unique names.

### Backup from SOURCE

By default `knife-ec-backup` will use a concurrency of 10.  We've found in testing that a concurrency of 50 seems to be a sweet spot and yields significant improvement.  Any higher than 50 and you will likely reach a point of diminishing returns.  Also, it is important to keep in mind that concurrency can cause a spike in the load on your Chef Server.  Thus, keep a close watch on your Chef Server stats while running a backup and terminate the backup if returning excessive HTTP 500s to chef client agents.

**IMPORTANT NOTE:** To add additional concurrency threads, append this to the following command: `--concurrency 50`

```
cd ../..
knife ec -c chef_backups/conf/knife_src_server.rb backup chef_backups --webui-key chef_backups/conf/webui_priv_src.pem
```

The command above will download all data from the source Chef Server and store objects as individual json files beneath `chef_backups`.  It is safe to re-run the backup multiple times over the existing `chef_backups` directory.  On subsequent runs, `knife-ec-backup` will do a differential backup of the `/cookbooks` objects, which potentially consume the most amount of time to transfer, in addition to the `nodes` objects.

### Restore to new DESTINATION

Again, for best results, the recommended strategy is to utilize a new destination cluster without any pre-existing data.

```
knife ec -c chef_backups/conf/knife_dst_server.rb restore chef_backups --webui-key chef_backups/conf/webui_priv_dst.pem
```

**IMPORTANT NOTE:** Unlike a backup operation, the default concurrency of 10 seems to yield the best results on a restore.  Bumping up the concurrency just slowed down the process in our testing.

If `knife-ec-backup` receives a 500 response while restoring, it will retry the operation up to 5 times with ever increasing delay between retries.  After the 5th failed attempt, the job will abort.  As with a backup, it is safe to re-run the restore multiple times to get a complete run.

### Testing Results
During our testing we set up an environment with the following:
- Standalone Chef Server (source) with 1,600 cookbooks and 10,000 nodes
- Backended HA Cluster (destination) consisting of 3 backends and 2 frontends
- The ec-backup of this configuration resulted in 1GB of total data

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
- Modifying the behavior of `knife-ec-backup` to incorporate the strategy in the above workaround: Just sending a POST and if 409, send a PUT.
- Adding the capability of uploading a tarball for both cookbooks and nodes and just letting the Chef Server handle it
- Modifying the Chef Server API so that there is a way to query if a node exists without having to transfer the entire node object.  This is of lesser value to the use-case scenario of restoring to a blank destination - the gratuitous POST is likely the best option.
- A variation of the above would be to modify the GET request on nodes to just request HEAD so that the full node object isn't returned in the request.
