# Example deployment scenarios

There are a four basic deployment scenarios that are supported by the playbooks in this repository. In the first two (shown below) we'll walk through the deployment of Zookeeper to a single node and the deployment of a multi-node Zookeeper ensemble using a static inventory file. In the third scenario we will show how the same multi-node Zookeeper ensemble deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file. Finally, in the last scenario we'll walk through the process of "growing" an existing Zookeeper ensemble by adding nodes to it.

## Scenario #1: deploying Zookeeper to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Zookeeper to a single node is really only only useful for very simple test environments. Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy Zookeeper to a single node with the IP address "192.168.34.12", we could simply create a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.12 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

[zookeeper]
192.168.34.12

$ 
```

Note that in this example inventory file the `ansible_ssh_host` and `ansible_ssh_port` will take their default values since they aren't specified for our host in this very simple static inventory file. Once we've built our static inventory file, we can then deploy Zookeeper to our single node by running an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory provision-zookeeper.yml
```

This will download the Apache Zookeeper distribution file from the default download server defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file, unpack that gzipped tarfile into the `/opt/apache-zookeeper` directory on that host, and configure that node as a single-node Zookeeper deployment, using the default configuration parameters that are defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file.

## Scenario #2: deploying a multi-node Zookeeper ensemble
If you are using this playbook to deploy a multi-node Zookeeper ensemble, then the configuration only slightly more complicated. The Zookeeper ensemble consists of a set of nodes that are configured to work together so that if any one node is lost the ensemble can continue to function.

Let's assume that we are deploying Zookeeper to an ensemble made up of three nodes and, furthermore, let's assume that we're going to be using a static inventory file to control this deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.18 ansible_ssh_host= 192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host= 192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host= 192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.21 ansible_ssh_host= 192.168.34.21 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.22 ansible_ssh_host= 192.168.34.22 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

$
```

To deploy Zookeeper to the three nodes in our static inventory file and configure those nodes to work together as an ensemble, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      data_iface: eth0, zookeeper_data_dir: '/data', \
      zookeeper_url: 'apache-zookeeper/zookeeper-3.4.9.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos' \
    }" provision-zookeeper.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
inventory_type: static
data_iface: eth0
zookeeper_url: 'apache-zookeeper/zookeeper-3.4.9.tar.gz'
yum_repo_url: 'http://192.168.34.254/centos'
zookeeper_data_dir: '/data'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look somethin like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }" provision-zookeeper.yml
```

As an aside, it should be noted here that the [provision-zookeeper.yml](../provision-zookeeper.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was shown above could also be run as:

```bash
$ ./provision-zookeeper.yml -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }"
```

This form is available as a replacement for any of the `ansible-playbook` commands that we show here; which form you use will likely be a matter of personal preference (since both accomplish the same thing).

Once that playbook run is complete, we can run the `zkServer.sh status` command on each of the nodes in our ensemble. This will show whether Zookeeper is running or not on each node and whether it is acting as a leader or follower:

```bash
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.18 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.19 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.20 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: leader
$ 
```

This `ansible-playbook` command deployed Zookeeper to all three of our nodes and configured them as a single ensemble using a static inventory file to control the deployment.

## Scenario #3: deploying a Zookeeper ensemble via dynamic inventory
In this section we will repeat the multi-node ensemble deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Zookeeper ensemble to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be configuring as an ensemble with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `zookeeper`
* Once all of the nodes that will make up our ensemble have been tagged appropriately, we can run `ansible-playbook` command similar to the first `ansible-playbook` command shown in the previous scenario; this will deploy Zookeeper to the nodes that make up our ensemble.

In terms of what the commands look like, lets assume for this example that we've tagged the nodes that we are going to use to build our Zookeeper ensemble with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: zookeeper

The `ansible-playbook` command used to deploy Zookeeper to target nodes in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -e "{ \
        application: zookeeper, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, \
        zookeeper_data_dir: '/data' \
    }" provision-zookeeper.yml
```

In an AWS environment, the `ansible-playbook` command looks quite similar:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: zookeeper, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, \
        zookeeper_data_dir: '/data' \
    }" provision-zookeeper.yml
```

As you can see, these two commands only in terms of the environment variable defined at the beginning of the command-line used to provision to the AWS environment (`AWS_PROFILE=datanexus_west`) and the value defined for the `cloud` variable (`osp` versus `aws`). In both cases the result would be a set of nodes deployed as a Zookeeper ensemble. The number of nodes in the ensemble will be determined (completely) by the number of nodes in the OpenStack or AWS environment that have been tagged with a matching set of `application`, `tenant`, `project` and `domain` tags.

## Scenario #4: adding nodes to a multi-node Zookeeper ensemble
When adding nodes to an existing Zoopeeker ensemble, we must be careful of several things:

* We don't want to redeploy Zookeeper to the existing nodes in the ensemble, only to the new nodes we are adding
* We want to make sure the nodes we are adding to the ensemble are configured properly to join that ensemble, but we don't want the `zookeeper` service started immediately after the deployment process is complete. Once we finish provisioning the the nodes that we want to add to the ensemble, we will have to:
    * Reconfigure the existing nodes so that the new nodes are included in the list of ensemble members maintained in the `{{zookeeper_dir}}/conf/zoo.cfg` file on each host. During this reconfiguration process we must be careful to ensure that the existing nodes keep their existing `broker.id` values. In addition, we must ensure that the new nodes that are being added are configured with unique `broker.id` values of their own (i.e. they are not assigned `broker.id` values that are already being used by the existing nodes in the ensemble)
    * Once all of the nodes have been (re)configured and list all of servers in the ensemble in their `{{zookeeper_dir}}/conf/zoo.cfg` file (both existing and new nodes) we must d etermine the role that each of the *existing nodes* plays in the ensemble (whether that node is a `follower` or `leader` node)
    * Once we've determined the roles that each of the *existing nodes* plays, we can start a rolling restart of the `zookeeper` services on each of those nodes; first restarting the `zookeeper` service on each of the follower nodes, sequentially, then restarting the `zookeeper` service on the `leader` node. We also need to pause between each node in the rolling restart process in order to ensure that each node has a chance to rejoin the ensemble before restarting the next node in the sequence
    * Once all of the existing nodes in the ensemble have been restarted, we can then go back to the new nodes that we are adding to the ensemble and start the `zookeeper` process on those nodes

To make this process as simple as possible (and ensure that there is no danger of reprovisioning the nodes in the existing ensemble when attempting to add new nodes to it), we have actually separated out the plays that are used to add nodes to an existing ensemble into a separate playbook (the [add-nodes.yml](./add-nodes.yml) file in this repository).

It is critical that the same configuration parameters be passed in during the process of adding new nodes to the ensemble as were passed in when building the ensemble initially. Zookeeper is not very tolerant of differences in configuration between members of a ensemble, so we will want to avoid those situations. The easiest way to manage this is to use a *local inventory file* to manage the configuration parameters that are used for a given ensemble, then pass in that file as an argument to the `ansible-playbook` command that you are running to add nodes to that ensemble. That said, in the dynamic inventory examples we show (below) we will define the configuration parameters that were set to non-default values in the previous playbook runs as extra variables that are passed into the `ansible-playbook` command on the command-line for clarity.

To provide a couple of examples of how this process of growing a ensemble works, we would first like to walk through the process of adding two new nodes to the existing Zookeeper ensemble that was created using the `test-cluster-inventory` (static) inventory file, above. The first step would be to edit the static inventory file and add the two new nodes to the `zookeeper` host group, then save the resulting file. The host group defined in the `test-cluster-inventory` file shown above would look like this after those edits:

```
[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20
192.168.34.21
192.168.34.22
```

(note that we have only shown the tail of that file; the hosts defined at the start of the file would remain the same). With the new static inventory file in place, the playbook command that we would run to add the two new nodes listed in the updated inventory file to our existing ensemble would look something like this:

```bash
$ ./add-nodes.yml -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }"
```

As you can see, this is essentially the same command we ran previously to provision our ensemble initially in the static inventory scenario. The only change to the previous command are that we are using a different playbook (the [add-nodes.yml](../add-nodes.yml) playbook instead of the [provision-zookeeper.yml](../provision-zookeeper.yml) playbook).

To add new nodes to an existing Zookeeper ensemble in an AWS or OpenStack environment, we would simply create the new nodes we want to add in that environment and tag them appropriately (using the same `Tenant`, `Application`, `Project`, and `Domain` tags that we used when creating our initial ensemble). With those new machines tagged appropriately, the command used to add a new set of nodes to an existing ensemble in an OpenStack environment would look something like this:

```bash
$ ansible-playbook -e "{ \
        application: zookeeper, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, \
        zookeeper_data_dir: '/data' \
    }" add-nodes.yml
```

The only difference when adding nodes to an AWS environment would be the environment variable that needs to be set at the beginning of the command-line (eg. `AWS_PROFILE=datanexus_west`) and the cloud value that we define within the extra variables that are passed into that `ansible-playbook` command (`aws` instead of `osp`):

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: zookeeper, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, \
        zookeeper_data_dir: '/data' \
    }" add-nodes.yml
```

As was the case with the static inventory example shown above, the command shown here for adding new nodes to an existing ensemble in an AWS or OpenStack cloud (using tags and dynamic inventory) is essentially the same command that was used when deploying the initial ensemble, but we are using a different playbook (the [add-nodes.yml](../add-nodes.yml) playbook instead of the [provision-zookeeper.yml](../provision-zookeeper.yml) playbook).
