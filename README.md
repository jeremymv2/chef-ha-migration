# Two weeks playing with `knife-ec-backup`, testing a Data Migration Strategy to Chef HA Backend Cluster

## Overview
Recently, I've focused on using `knife-ec-backup` with the goal of creating a plan to assist data migration from existing Chef topologies to Chef HA backend cluster.  Here are some notes on what I've discovered.

`knife-ec-backup` is Chef's tool for migrating data between Chef server clusters of varying topologies (Standalone, Tier, old-HA, new-HA) and versions (with some limitations).   Unlike Chef's filesystem-based backup/restore tools (`pg_dumpall` or `chef-server-ctl backup|restore`), it is going to be more flexible. Filesystem-based backup/restore tools can ONLY be used to backup/restore LIKE versions and LIKE topologies of Chef Server. `knife-ec-backup` on the other hand, creates a highly portable object-based backup of the entire Chef server including all Orgs, Users, Cookbooks, Databags, etc.

Because Chef HA does not have an in-place migration tool, the expectation is that you will build and validate a new Chef HA Cluster and then migrate your data to it.

The steps to building a new Chef HA Cluster are as follows:

1. Build a new Chef HA Cluster (bonus points for using [automation](https://github.com/ncerny/chef_stack) to reproducibly build your cluster)
2. Validate your new Chef HA Cluster  (we cannot stress enough the importance of this)
    * Test each new frontend using `chef-server-ctl test`
    * Test both hard and soft failovers of the backend systems
    * Load test your new cluster using [chef-load](https://github.com/jeremiahsnapp/chef-load)
3. Perform a full backup of your current production Chef Server cluster.  Note:
    * `knife-ec-backup` can generate considerable load on your cluster, particularly when increasing parallelism.
    * It is recommend that you upgrade to the latest versions of Chef Server if at all possible.   Using the latest version of `knife-ec-backup` is a requirement.
    * If you begin to experience 500 errors on your existing Chef Server during a full backup, attempt these during off-peak hours if possible.
    * Performance tuning of your Chef server may be required.  For more information see:
        * [Monitoring and Tuning your Chef Server](https://www.slideshare.net/AndrewDuFour1/monitoring-and-tuning-your-chef-server-chef-conf-talk)
        * [Tuning the Chef Server for Scale](http://irvingpop.github.io/blog/2015/04/20/tuning-the-chef-server-for-scale/)
        * [Understanding the Chef Server](https://www.youtube.com/watch?v=22GtVMHJDsI)
    * An advanced strategy may be to temporarily add a dedicated Frontend to existing Tier/HA topologies in order to reduce loading on the remaining frontends
4. Perform a full restore to the new cluster
    * The performance monitoring and tuning advice from the previous step will help achieve higher levels of parallelism
    * There may be errors encountered during restore - for example expired user-org invitations that point to deleted users.  It is recommended that you fix as many of these as possible on the source server.   If that's not possible, it's possible to fix the errors directly in the JSON filess as an intermediate "data cleanup" stage before restoring.
5. Validate the target cluster
    * Retest the cluster both `chef-server-ctl test` as well as re-pointing a number of non-critical nodes at the new cluster
6. Perform nightly incremental backups/restores
    * `knife-ec-backup` can operate in a pseudo-incremental mode as long as you keep the backup directory intact.  Continue to run backups/restores and you'll notice they complete much faster than the original
    * Note the time it takes for an incremental backup/restore to complete - this should provide you with the clearest guidance for how long of a downtime/maintenance window to schedule
    * An advanced strategy is to migrate one org or batches of orgs at a time.  In this case you'll need to:
        * Use the chef-client cookbook or similar strategy to update the client.rb file on every node to point at the new cluster
        * Filter already-migrated orgs from the backup/restore once migration is complete


### Initial Setup

Use a dedicated backup system other than your existing Chef Server.  You will need to install the most recent `knife-ec-bacup` and
should probably not install that on your existing Chef Server.

On a dedicated backup system/workstation, install the [ChefDK](https://downloads.chef.io/chefdk)

Install latest knife-ec-backup gem from rubygems.org
```bash
chef gem install knife-ec-backup
```

Or install from master if you need the absolute latest:
```bash
git clone git@github.com:chef/knife-ec-backup.git
cd knife-ec-backup
chef gem build knife-ec-backup.gemspec
chef gem install knife-ec-backup*gem --no-ri --no-rdoc -V
```

Create a local backup working directory/folder and config file location
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

### Backup the Source

By default `knife-ec-backup` will use a concurrency of 10.  We've found in testing that a concurrency of 50 seems to be a sweet spot and yields significant improvement.  Any higher than 50 and you will likely reach a point of diminishing returns.  Also, it is important to keep in mind that concurrency can cause a spike in the load on your Chef Server.  Thus, keep a close watch on your Chef Server stats while running a backup and terminate the backup if returning excessive HTTP 500s to chef client agents.

Recently the `--purge` option was added to `knife-ec-backup`.  This command line option controls whether to sync deletions from backup source to restore destination. (default: false).  If you are running multiple backups then consider using `--purge` so that things such as user deletions will be synced.

**IMPORTANT:** To add additional concurrency threads, append `--concurrency 50` to your backup command.

**IMPORTANT:** To sync deletions, append `--purge` to your backup command.

```bash
cd ../..
knife ec -c chef_backups/conf/knife_src_server.rb backup chef_backups --webui-key chef_backups/conf/webui_priv_src.pem
```

The command above will download all data from the source Chef Server and store objects as individual json files beneath `chef_backups`.  It is safe to re-run the backup multiple times over the existing `chef_backups` directory.  On subsequent runs, `knife-ec-backup` will do a differential backup of the `/cookbooks` objects.

**Note:** If you are using the Chef Manage addon and your users are not in ldap and you wish to preserve their passwords, then you will need to also specify `--with-user-sql` as indicated [here](https://github.com/chef/knife-ec-backup#knife-ec-backup-dest_dir-options).

**Note:** Because the `backup` operation can be run multiple times and only NEW cookbooks added to the source will be transferred down, one good strategy may be to run repeated backups ahead of the migration day if the initial backup takes a prohibitively long time. Running several small backups ahead of time may be better than running one BIG one. Another supporting strategy might be performing backups during low-peak times, or adding frontend capacity during both backups (db connections) and restores (frontend CPU bound).

### Restore to the new Chef HA Cluster

Again, for best results, the recommended strategy is to utilize a new destination cluster without any pre-existing data.

**IMPORTANT:** To sync any deletions, append `--purge` to your restore command.

```bash
knife ec -c chef_backups/conf/knife_dst_server.rb restore chef_backups --webui-key chef_backups/conf/webui_priv_dst.pem
```

**IMPORTANT NOTE:** Unlike a backup operation, the default concurrency of 10 seems to yield the best results on a restore.  Bumping up the concurrency any further will not produce any gains.

If `knife-ec-backup` receives a 500 response while restoring, it will retry the operation up to 5 times with ever increasing delay between retries.  After the 5th failed attempt, the job will abort.  As with a backup, it is safe to re-run the restore multiple times to get a complete run.

### Scenario used for Testing
During our testing we set up an environment with the following, utilizing an implementation of the resource cookbook [chef_stack](https://github.com/ncerny/chef_stack):
- Standalone Chef Server (source) with 1,600 cookbook versions and 10,000 nodes
- Backended HA Cluster (destination) consisting of 3 backends and 2 frontends
- The ec-backup of this configuration resulted in 1GB of total data stored on the workstation.

### Discoveries
1. Bumping the concurrency setting significantly improves backup times while having a detrimental effect on restores.
2. Cookbook backup and restores are incremental and only transferred if the src/dst is different.
3. Backup/Restore has not been optimized yet for large user sets.  If you have a large user set (eg. > 1,000) this can add ~15 mins.

### Optimizations made recently (March 2017):
- https://github.com/chef/knife-ec-backup/pull/92
- https://github.com/chef/knife-ec-backup/pull/88
- https://github.com/chef/knife-ec-backup/pull/87
- https://github.com/chef/chef/pull/5890

An example knife-ec-backup shell script can be found in this [gist](https://gist.github.com/jeremymv2/9c14086e13cac572235fe4bab5881ed3)
- Additional information in the `knife-ec-backup` [README.md](https://github.com/chef/knife-ec-backup/blob/master/README.md)
