---
type: reference
---

# How to set up Consul **(PREMIUM ONLY)**

A Consul cluster consists of both
[server and client agents](https://www.consul.io/docs/agent).
The servers run on their own nodes and the clients run on other nodes that in
turn communicate with the servers.

GitLab Premium includes a bundled version of [Consul](https://www.consul.io/)
a service networking solution that you can manage by using `/etc/gitlab/gitlab.rb`.

## Configure the Consul nodes

NOTE: **Important:**
Before proceeding, refer to the
[available reference architectures](reference_architectures/index.md#available-reference-architectures)
to find out how many Consul server nodes you should have.

On **each** Consul server node perform the following:

1. Follow the instructions to [install](https://about.gitlab.com/install/)
   GitLab by choosing your preferred platform, but do not supply the
   `EXTERNAL_URL` value when asked.
1. Edit `/etc/gitlab/gitlab.rb`, and add the following by replacing the values
   noted in the `retry_join` section. In the example below, there are three
   nodes, two denoted with their IP, and one with its FQDN, you can use either
   notation:

   ```ruby
   # Disable all components except Consul
   roles ['consul_role']

   # Consul nodes: can be FQDN or IP, separated by a whitespace
   consul['configuration'] = {
     server: true,
     retry_join: %w(10.10.10.1 consul1.gitlab.example.com 10.10.10.2)
   }

   # Disable auto migrations
   gitlab_rails['auto_migrate'] = false
   ```

1. [Reconfigure GitLab](restart_gitlab.md#omnibus-gitlab-reconfigure) for the changes
   to take effect.
1. Run the following command to ensure Consul is both configured correctly and
   to verify that all server nodes are communicating:

   ```shell
   sudo /opt/gitlab/embedded/bin/consul members
   ```

   The output should be similar to:

   ```plaintext
   Node                 Address               Status  Type    Build  Protocol  DC
   CONSUL_NODE_ONE      XXX.XXX.XXX.YYY:8301  alive   server  0.9.2  2         gitlab_consul
   CONSUL_NODE_TWO      XXX.XXX.XXX.YYY:8301  alive   server  0.9.2  2         gitlab_consul
   CONSUL_NODE_THREE    XXX.XXX.XXX.YYY:8301  alive   server  0.9.2  2         gitlab_consul
   ```

   If the results display any nodes with a status that isn't `alive`, or if any
   of the three nodes are missing, see the [Troubleshooting section](#troubleshooting-consul).

## Upgrade the Consul nodes

To upgrade your Consul nodes, upgrade the GitLab package.

Nodes should be:

- Members of a healthy cluster prior to upgrading the Omnibus GitLab package.
- Upgraded one node at a time.

Identify any existing health issues in the cluster by running the following command
within each node. The command will return an empty array if the cluster is healthy:

```shell
curl http://127.0.0.1:8500/v1/health/state/critical
```

Consul nodes communicate using the raft protocol. If the current leader goes
offline, there needs to be a leader election. A leader node must exist to facilitate
synchronization across the cluster. If too many nodes go offline at the same time,
the cluster will lose quorum and not elect a leader due to
[broken consensus](https://www.consul.io/docs/internals/consensus.html).

Consult the [troubleshooting section](#troubleshooting-consul) if the cluster is not
able to recover after the upgrade. The [outage recovery](#outage-recovery) may
be of particular interest.

NOTE: **Note:**
GitLab uses Consul to store only transient data that is easily regenerated. If
the bundled Consul was not used by any process other than GitLab itself, then
[rebuilding the cluster from scratch](#recreate-from-scratch) is fine.

## Troubleshooting Consul

Below are some useful operations should you need to debug any issues.
You can see any error logs by running:

```shell
sudo gitlab-ctl tail consul
```

### Check the cluster membership

To determine which nodes are part of the cluster, run the following on any member in the cluster:

```shell
sudo /opt/gitlab/embedded/bin/consul members
```

The output should be similar to:

```plaintext
Node            Address               Status  Type    Build  Protocol  DC
consul-b        XX.XX.X.Y:8301        alive   server  0.9.0  2         gitlab_consul
consul-c        XX.XX.X.Y:8301        alive   server  0.9.0  2         gitlab_consul
consul-c        XX.XX.X.Y:8301        alive   server  0.9.0  2         gitlab_consul
db-a            XX.XX.X.Y:8301        alive   client  0.9.0  2         gitlab_consul
db-b            XX.XX.X.Y:8301        alive   client  0.9.0  2         gitlab_consul
```

Ideally all nodes will have a `Status` of `alive`.

### Restart Consul

If it is necessary to restart Consul, it is important to do this in
a controlled manner to maintain quorum. If quorum is lost, to recover the cluster,
you will need to follow the Consul [outage recovery](#outage-recovery) process.

To be safe, it's recommended that you only restart Consul in one node at a time to
ensure the cluster remains intact. For larger clusters, it is possible to restart
multiple nodes at a time. See the
[Consul consensus document](https://www.consul.io/docs/internals/consensus.html#deployment-table)
for how many failures it can tolerate. This will be the number of simultaneous
restarts it can sustain.

To restart Consul:

```shell
sudo gitlab-ctl restart consul
```

### Consul nodes unable to communicate

By default, Consul will attempt to
[bind](https://www.consul.io/docs/agent/options.html#_bind) to `0.0.0.0`, but
it will advertise the first private IP address on the node for other Consul nodes
to communicate with it. If the other nodes cannot communicate with a node on
this address, then the cluster will have a failed status.

If you are running into this issue, you will see messages like the following in `gitlab-ctl tail consul` output:

```plaintext
2017-09-25_19:53:39.90821     2017/09/25 19:53:39 [WARN] raft: no known peers, aborting election
2017-09-25_19:53:41.74356     2017/09/25 19:53:41 [ERR] agent: failed to sync remote state: No cluster leader
```

To fix this:

1. Pick an address on each node that all of the other nodes can reach this node through.
1. Update your `/etc/gitlab/gitlab.rb`

   ```ruby
   consul['configuration'] = {
     ...
     bind_addr: 'IP ADDRESS'
   }
   ```

1. Reconfigure GitLab;

   ```shell
   gitlab-ctl reconfigure
   ```

If you still see the errors, you may have to
[erase the Consul database and reinitialize](#recreate-from-scratch) on the affected node.

### Consul does not start - multiple private IPs

In case that a node has multiple private IPs, Consul will be confused as to
which of the private addresses to advertise, and then immediately exit on start.

You will see messages like the following in `gitlab-ctl tail consul` output:

```plaintext
2017-11-09_17:41:45.52876 ==> Starting Consul agent...
2017-11-09_17:41:45.53057 ==> Error creating agent: Failed to get advertise address: Multiple private IPs found. Please configure one.
```

To fix this:

1. Pick an address on the node that all of the other nodes can reach this node through.
1. Update your `/etc/gitlab/gitlab.rb`

   ```ruby
   consul['configuration'] = {
     ...
     bind_addr: 'IP ADDRESS'
   }
   ```

1. Reconfigure GitLab;

   ```shell
   gitlab-ctl reconfigure
   ```

### Outage recovery

If you lost enough Consul nodes in the cluster to break quorum, then the cluster
is considered failed, and it will not function without manual intervention.
In that case, you can either recreate the nodes from scratch or attempt a
recover.

#### Recreate from scratch

By default, GitLab does not store anything in the Consul node that cannot be
recreated. To erase the Consul database and reinitialize:

```shell
sudo gitlab-ctl stop consul
sudo rm -rf /var/opt/gitlab/consul/data
sudo gitlab-ctl start consul
```

After this, the node should start back up, and the rest of the server agents rejoin.
Shortly after that, the client agents should rejoin as well.

#### Recover a failed node

If you have taken advantage of Consul to store other data and want to restore
the failed node, follow the
[Consul guide](https://learn.hashicorp.com/consul/day-2-operations/outage)
to recover a failed cluster.