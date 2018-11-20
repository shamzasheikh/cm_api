---
layout: default
title: Python Client (Deprecated)
id: python-client
permalink: /docs/python-client/
---

#### Table of Contents ####

* toc
{:toc}


Version
=======
From Cloudera Manager version 6.0 i.e. API version 30 onwards, this Python API client
has been deprecated.

Use new [Swagger based python API client]({{ site.url }}/docs/python-client-swagger/)
for Cloudera Manager version 6.0 i.e. API version 30 onwards.

Installation
============
**To install the Python API client, simply:**

    $ sudo pip install cm-api

If your system does not have `pip`, you can get it from your distro:

    $ sudo apt-get install python-pip
    ## ... or use `yum install` if you are on CentOS

**Alternatively, you can also install from source:**

    $ git clone git://github.com/cloudera/cm_api.git
    $ cd cm_api/python
    $ sudo python setup.py install
    ## ... the distribution egg file is in ./dist/

**Once you have run one of the above commands, you can check to see
if the install succeeded by running** `pip show cm-api`.

Epydoc
======
Here is the last [epydoc]({{ site.url }}/epydoc/5.14.0/index.html) with older
python API client, for API version 19 (CM 5.14.0)


Basic Usage
===========
Each subsection continues from the previous one.

{% highlight python %}
# Get a handle to the API client
from cm_api.api_client import ApiResource

cm_host = "cm-host"
api = ApiResource(cm_host, username="admin", password="admin")

# Get a list of all clusters
cdh4 = None
for c in api.get_all_clusters():
  print c.name
  if c.version == "CDH4":
    cdh4 = c

## -- Output --
# Cluster 1 - CDH4
# Cluster 2 - CDH3
{% endhighlight %}


The `ApiResource` constructor takes other optional arguments, to specify
the API version, CM server port, HTTPS vs HTTP, etc.


Inspecting a Service
--------------------

Now we have identified a CDH4 cluster. Find the HDFS service:

{% highlight python %}
for s in cdh4.get_all_services():
  print s
  if s.type == "HDFS":
    hdfs = s

## -- Output --
# <ApiService>: ZOOKEEPER-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: HDFS-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: HBASE-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: MAPREDUCE-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: YARN-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: OOZIE-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: HIVE-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: HUE-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: IMPALA-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: SOLR-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: KS_INDEXER-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: SQOOP-1 (cluster: Cluster 1 - CDH4)
# <ApiService>: FLUME-1 (cluster: Cluster 1 - CDH4)
{% endhighlight %}

Inspect the HDFS service health and status:

{% highlight python %}
print hdfs.name, hdfs.serviceState, hdfs.healthSummary
## -- Output --
# HDFS-1 STARTED GOOD

print hdfs.serviceUrl
## -- Output --
# http://cm-host:7180/cmf/serviceRedirect/HDFS-1

for chk in hdfs.healthChecks:
  print "%s --- %s" % (chk['name'], chk['summary'])

## -- Output --
# HDFS_BLOCKS_WITH_CORRUPT_REPLICAS --- GOOD
# HDFS_CANARY_HEALTH --- GOOD
# HDFS_DATA_NODES_HEALTHY --- GOOD
# HDFS_FREE_SPACE_REMAINING --- GOOD
# HDFS_HA_NAMENODE_HEALTH --- GOOD
# HDFS_MISSING_BLOCKS --- GOOD
# HDFS_STANDBY_NAMENODES_HEALTHY --- DISABLED
# HDFS_UNDER_REPLICATED_BLOCKS --- GOOD
{% endhighlight %}

Inspecting a Role
-----------------

Find the NameNode and get basic info:

{% highlight python %}
nn = None
for r in hdfs.get_all_roles():
  if r.type == 'NAMENODE':
    nn = r
print "Role name: %s\nState: %s\nHealth: %s\nHost: %s" % (
    nn.name, nn.roleState, nn.healthSummary, nn.hostRef.hostId)

## -- Output --
# Role name: HDFS-1-NAMENODE-03d442721157b71e72b7469113f254f5
# State: STOPPED
# Health: GOOD
# Host: nightly47-1.ent.cloudera.com
{% endhighlight %}

Similar to the service example, roles also expose their health checks.


Getting Metrics
---------------

First we look at what metrics are available:

{% highlight python %}
# NOTE: this does not work starting in v6 of the api (CM5.0.0). Use the
# timeseries endpoint dicussed below or set your api version to v5.
metrics = hdfs.get_metrics()
print len(metrics)
## -- Output --
# 651

for m in metrics:
  print "%s (%s)" % (m.name, m.unit)

## -- Output --
# fsync_nanos_avg_time_datanode_min (nanos)
# fsync_nanos_avg_time_datanode_weighted_std_dev (nanos)
# files_total (blocks)
# excess_blocks (blocks)
# block_capacity (blocks)
# pending_replication_blocks (blocks)
# scheduled_replication_blocks (blocks)
# rpc_processing_time_time_datanode_sum (ms)
# rpc_processing_time_avg_time_datanode_max (ms)
# rpc_processing_time_avg_time_datanode_min (ms)
# rpc_processing_time_avg_time_datanode_weighted_std_dev (ms)
# rpc_processing_time_avg_time_datanode_weighted_avg (ms)
# events_important (events)
# heartbeats_num_ops_datanode_std_dev_rate (operations per second)
# total_load (transceivers)
# fsync_nanos_time_datanode_sum (nanos)
# expired_heartbeats (heartbeats)
# heartbeats_num_ops_datanode_min_rate (operations per second)
#     ... omitting the other 600+ metrics
{% endhighlight %}

Reading a metric. Suppose we are interested in the `files_total` and
`dfs_capacity_used` metrics, over the last 30 minutes.

{% highlight python %}
import time
import datetime

from_time = datetime.datetime.fromtimestamp(time.time() - 1800)
to_time = datetime.datetime.fromtimestamp(time.time())
query = "select files_total, dfs_capacity_used " \
        "where serviceName = HDFS-1 " \
        "  and category = SERVICE"

result = api.query_timeseries(query, from_time, to_time)
ts_list = result[0]
for ts in ts_list.timeSeries:
  print "--- %s: %s ---" % (ts.metadata.entityName, ts.metadata.metricName)
  for point in ts.data:
    print "%s:\t%s" % (point.timestamp.isoformat(), point.value)

## -- Output --
# --- HDFS-1: files_total ---
# 2013-09-04T22:12:34.983000:     157.0
# 2013-09-04T22:13:34.984000:     157.0
# 2013-09-04T22:14:34.984000:     157.0
#     ... omitting a bunch of values
# 
# --- HDFS-1: dfs_capacity_used ---
# 2013-09-04T22:12:34.983000:     186310656.0
# 2013-09-04T22:13:34.984000:     186310656.0
# 2013-09-04T22:14:34.984000:     186310656.0
#     ... omitting a bunch of values
{% endhighlight %}

This example uses the new-style `/cm/timeseries` endpoint (which uses
[tsquery](http://tiny.cloudera.com/tsquery)) to get metric data points.
Even though the example is querying HDFS metrics, the processing logic is the
same for all queries.

The old-style `.../metrics` endpoints (which exists under host, service
and role objects) are mostly useful for exploring what metrics are available.


Service Lifecycle and Commands
------------------------------

Restart HDFS. Start and stop work similarly:

{% highlight python %}
cmd = hdfs.restart()
print cmd.active
## -- Output --
# True

cmd = cmd.wait()
print "Active: %s. Success: %s" % (cmd.active, cmd.success)
## -- Output --
# Active: False. Success: True
{% endhighlight %}

You can also chain the `wait()` call:

{% highlight python %}Managing Parcels
cmd = hdfs.restart().wait()
{% endhighlight %}

Restart the NameNode. Commands on roles are issued at the service level,
and may be done in bulk.

{% highlight python %}
cmds = hdfs.restart_roles(nn.name)
for cmd in cmds:
  print cmd

## -- Output --
# <ApiCommand>: 'Restart' (id: 225; active: True; success: None)
{% endhighlight %}

Configuring Services and Roles
------------------------------
First, lets look at all possible service configs. For legacy reasons, this is a
2-tuple of service configs and an empty dictionary (as of API v3), so you have
to add "\[0\]" below.
{% highlight python %}
for name, config in hdfs.get_config(view='full')[0].items():
  print "%s - %s - %s" % (name, config.relatedName, config.description)

## -- Output -- (just one line of many)
# dfs_replication - dfs.replication - Default block replication. The number of replications to make when the file is created. The default value is used if a replication number is not specified.
{% endhighlight %}

Now let's change dfs_replication to 2. We use "dfs_replication" and not
dfs.replication" because we must match the keys of the config view. This is
also the same value as ApiConfig.name.

{% highlight python %}
hdfs.update_config({'dfs_replication': 2})
# returns current configs, excluding defaults. Same as hdfs.get_config()
## -- Output --
# ({u'dfs_block_local_path_access_user': u'impala', u'zookeeper_service': u'zookeeper', u'dfs_replication': u'2'}, {})
{% endhighlight %}

Configuring roles is done similarly. Normally you want to modify groups instead
of modifying each role one by one.

First, find the group(s).
{% highlight python %}
dn_groups = []
for group in hdfs.get_all_role_config_groups():
  if group.roleType == 'DATANODE':
    dn_groups.append(group)
{% endhighlight %}

See all possible role configuration. It's the same for all groups of the same
role type in clusters with the same CDH version.

{% highlight python %}
for name, config in dn_groups[0].get_config(view='full').items():
  print "%s - %s - %s" % (name, config.relatedName, config.description)

## -- Output -- (just one line of many)
# process_auto_restart -  - When set, this role's process is automatically (and transparently) restarted in the event of an unexpected failure.
{% endhighlight %}

Let's configure our data nodes to auto-restart.

{% highlight python %}
for group in dn_groups:
  group.update_config({'process_auto_restart': True})
# returns config summary for group.
## -- Output --
# {u'dfs_datanode_data_dir_perm': u'755', u'dfs_data_dir_list': u'/dfs/dn', u'process_auto_restart': u'true'}
{% endhighlight %}

To reset a config to default, pass in a value of None.

{% highlight python %}
for group in dn_groups:
  group.update_config({'process_auto_restart': None})
# note process_auto_restart is missing from return value now
## -- Output --
# {u'dfs_datanode_data_dir_perm': u'755', u'dfs_data_dir_list': u'/dfs/dn'}
{% endhighlight %}

Managing Parcels
------------------------------

These examples cover how to get a new parcel up and running on
a cluster. Normally you would pick a specific parcel repository
and parcel version you want to install.

Add a CDH parcel repository. Note that in CDH 4, Impala and Solr are
in separate parcels. They are included in the CDH 5 parcel.

These examples requires v5 of the CM API or higher.

{% highlight python %}
# replace parcel_repo with the parcel repo you want to use
parcel_repo = 'http://archive.cloudera.com/cdh4/parcels/4.3.2.2/'
cm_config = api.get_cloudera_manager().get_config(view='full')
repo_config = cm_config['REMOTE_PARCEL_REPO_URLS']
value = repo_config.value or repo_config.default
# value is a comma-separated list
value += ',' + parcel_repo
api.get_cloudera_manager().update_config({
  'REMOTE_PARCEL_REPO_URLS': value})
# wait to make sure parcels are refreshed
time.sleep(10)
{% endhighlight %}

Download the parcel to the CM server.

{% highlight python %}
# replace cluster_name with the name of your cluster
cluster_name = 'Cluster 1 - CDH4'
cluster = api.get_cluster(cluster_name)
# replace parcel_version with the specific parcel version you want to install
# After adding your parcel repository to CM, you can use the API to list all parcels and get the precise version string by inspecting:
# cluster.get_all_parcels() or looking at the URL http://<cm_host>:7180/api/v5/clusters/<cluster_name>/parcels/
parcel_version = '4.3.2-1.cdh4.3.2.p0.2'
parcel = cluster.get_parcel('CDH', parcel_version)
parcel.start_download()
# unlike other commands, check progress by looking at parcel stage and status
while True:
  parcel = cluster.get_parcel('CDH', parcel_version)
  if parcel.stage == 'DOWNLOADED':
    break
  if parcel.state.errors:
    raise Exception(str(parcel.state.errors))
  print "progress: %s / %s" % (parcel.state.progress, parcel.state.totalProgress)
  time.sleep(15) # check again in 15 seconds

print "downloaded CDH parcel version %s on cluster %s" % (parcel_version, cluster_name)
{% endhighlight %}

Distribute the parcel so all agents on that cluster have a local copy of the parcel.

{% highlight python %}
parcel.start_distribution()
while True:
  parcel = cluster.get_parcel('CDH', parcel_version)
  if parcel.stage == 'DISTRIBUTED':
    break
  if parcel.state.errors:
    raise Exception(str(parcel.state.errors))
  print "progress: %s / %s" % (parcel.state.progress, parcel.state.totalProgress)
  time.sleep(15) # check again in 15 seconds

print "distributed CDH parcel version %s on cluster %s" % (parcel_version, cluster_name)
{% endhighlight %}

Activate the parcel so services pick up the new binaries upon next restart.

{% highlight python %}
parcel.activate()
{% endhighlight %}

Restart your cluster to pick up the new parcel.

{% highlight python %}
cluster.stop().wait()
cluster.start().wait()
{% endhighlight %}

Cluster Template
------------------------------

These examples cover how to export and import cluster template.

These examples requires v12 of the CM API or higher.

Import following modules

{% highlight python %}
import json
from cm_api.api_client import ApiResource
from cm_api.endpoints.types import ApiClusterTemplate
from cm_api.endpoints.cms import ClouderaManager
{% endhighlight %}

Export the cluster template as a json file

{% highlight python %}
resource = ApiResource("source-host", 7180, "admin", "admin", version=12)
cluster = resource.get_cluster("Cluster 1")
template = cluster.export()
with open('/tmp/template.json', 'w') as outfile:
json.dump(template.to_json_dict(), outfile, indent=4, sort_keys=True)
{% endhighlight %}

Make the required changes in the template file manually or using the python api's. User needs to map the hosts in the target cluster with right host templates and provide information about all the variables,like database information in the target cluster.

{% highlight xml %}
  "instantiator" : {
    "clusterName" : "<changeme>",
    "hosts" : [ {
      "hostName" : "<changeme>",
      "hostTemplateRefName" : "<changeme>",
      "roleRefNames" : [ "HDFS-1-NAMENODE-18041ba96f26361b0735d72598476dc1" ]
    }, {
      "hostName" : "<changeme>",
      "hostTemplateRefName" : "<changeme>"
    }, {
      "hostNameRange" : "<HOST[0001-0002]>",
      "hostTemplateRefName" : "<changeme>"
    } ],
    "variables" : [ {
      "name" : "FLUME-1-flume_truststore_password",
      "value" : "<changeme>"
    }, {
      "name" : "HBASE-1-HBASERESTSERVER-BASE-hbase_restserver_keystore_keypassword",
      "value" : "<changeme>"
    }, {
    .
    .
{% endhighlight %}    

Invoking import cluster template on the target cluster

{% highlight python %}
resource = ApiResource("target_host", 7180, "admin", "admin", version=12)
with open('/tmp/template.json') as data_file:
  data = json.load(data_file)
template = ApiClusterTemplate(resource).from_json_dict(data, resource)
cms = ClouderaManager(resource)
command = cms.import_cluster_template(template)
{% endhighlight %}

User can use this command to track the progress. The progress can be tracked by command details page in UI 
