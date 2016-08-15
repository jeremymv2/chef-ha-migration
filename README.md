# Data Migration Strategy for HA Chef Backend
Migrating data from non-HA Chef Server to Chef HA Backend.

This assumes a newly installed Chef HA backend cluster is available and not yet populated with data.


#### Create an Organization on one of the Front End nodes of the new HA cluster

```
sudo chef-server-ctl user-create backup Bob Backup backup@example.com securepassword --filename backupkey.pem

sudo chef-server-ctl org-create brewinc "Brew, Inc." --association_user admin --filename brewinc-validator.pem
```

#### Set up your work environment

Install latest knife-ec-backup gem and create workspace for backups
```
chef gem install knife-ec-backup
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

Copy the `/etc/opscode/webui_priv.pem` files from SRC and DST Chef Servers locally into `chef_backups/conf`

Backup from SRC
```
cd ../..
knife ec -c chef_backups/conf/knife_src_server.rb backup chef_backups --webui-key chef_backups/conf/webui_priv_src.pem
```

The command above will download all data from the src Chef Server and store them as json objects in `chef_backups`

Import the data into DST HA Cluster
```
knife ec -c chef_backups/conf/knife_src_server.rb restore chef_backups --webui-key chef_backups/conf/webui_priv_dst.pem
```
