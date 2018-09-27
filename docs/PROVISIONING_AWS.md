# OpenShift on AWS EC2 using CASL

## Local Setup (one time, only)

> **NOTE:** These steps are a canned set of steps serving as an example, and may be different in your environment.

In addition to _cloning this repo_, you'll need the following:

* Access to an AWS account with the proper policies to create resources (see details below)
* Docker installed
  * RHEL/CentOS: `yum install -y docker`
  * Fedora: `dnf install -y docker`
  * **NOTE:** If you plan to run docker as yourself (non-root), your username must be added to the `docker` user group.

* Although not strictly necessary, it is very handy to have the aws cli installed
  * `pip install awscli --upgrade --user`
  * More info [here](https://docs.aws.amazon.com/cli/latest/userguide/awscli-install-linux.html)


```shell
cd ~/src/
git clone https://github.com/redhat-cop/casl-ansible.git
```

* Run `ansible-galaxy` to pull in the necessary requirements for the CASL provisioning of OpenShift on AWS:

> **NOTE:** The target directory ( `galaxy` ) is **important** as the playbooks know to source roles and playbooks from that location.

```shell
cd ~/src/casl-ansible
ansible-galaxy install -r casl-requirements.yml -p galaxy
```

## AWS specific requirements
* Available parameters to use the AWS provision can be found in the Role's [README](../roles/manage-aws-infra/README.md)
* A [Key-pair in AWS](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair)
* Modify 'regions' entry (line 13) in the inventory 'ec2.ini' file to match the 'aws_region' variable in your inventory
* Modify 'instance_filters' entry (line 14) in the inventory 'ec2.ini' file to match the 'env_id' variable in your inventory's `all.yml`

Cool! Now you're ready to provision OpenShift clusters on AWS

## Provision an OpenShift Cluster

As an example, we'll provision the `sample.aws.example.com` cluster defined in the `~/src/casl-ansible/inventory` directory.

> **NOTE**: *It is recommended that you use a different inventory similar to the ones found in the `~src/casl-ansible/inventory` directory and keep it elsewhere. This allows you to update/remove/change your casl-ansble source directory without losing your inventory. Also note that it may take some effort to get the inventory just right, hence it is very beneficial to keep it around for future use without having to redo everything.*

The following is just an example on how the `sample.aws.example.com` inventory can be used:

1) Edit `~/src/casl-ansible/inventory/sample.aws.example.com.d/inventory/group_vars/all.yml` to match your AWS environment. See comments in the file for more detailed information on how to fill these in.

Typically you would need to modify:

```yaml
env_id
cloud_infrastructure
aws_access_key #If not using env vars
aws_secret_key #If not using env vars
dns_domain
rhsm* # rhsm vars according to your case (AK, Sat6, etc)
```

> **NOTE**: To know the available AZs of the region you are deploying in, run this aws cli command: `aws ec2 describe-availability-zones --region <REPLACE WITH AWS REGION>`  

2) Edit `~/src/casl-ansible/inventory/sample.aws.example.com.d/inventory/group_vars/OSEv3.yml` for your AWS specific configuration. See comments in the file for more detailed information on how to fill these in.

Typically you would need to modify:

```yaml
aws_access_key #If not using env vars
aws_secret_key #If not using env vars
```

3) Run the `end-to-end` provisioning playbook via our [AWS installer container image](../images/installer-aws/).

```shell
docker run -u `id -u` \
      -v <REPLACE WITH AWS KEYPAIR FILE PATH>:/opt/app-root/src/.ssh/id_rsa:Z \
      -v <REPLACE WITH SRC FOLDER PATH>:/tmp/src:Z \ # Folder where casl-ansible was cloned
      -e INVENTORY_DIR=/tmp/src/casl-ansible/inventory/sample.aws.example.com.d/inventory \
      -e PLAYBOOK_FILE=/tmp/src/casl-ansible/playbooks/openshift/end-to-end.yml \
      -e OPTS="-e aws_key_name=<REPLACE WITH AWS KEYPAIR NAME>" -t \
      redhatcop/installer-aws
```

For example:

```shell
docker run -u `id -u` \
      -v $HOME/.ssh/id_rsa:/opt/app-root/src/.ssh/id_rsa:Z \
      -v $HOME/src/:/tmp/src:Z \
      -e INVENTORY_DIR=/tmp/src/casl-ansible/inventory/sample.aws.example.com.d/inventory \
      -e PLAYBOOK_FILE=/tmp/src/casl-ansible/playbooks/openshift/end-to-end.yml \
      -e OPTS="-e aws_key_name=my-key-name" -t \
      redhatcop/installer-aws
```

> **NOTE 1:** The `aws_key_name` variable at the end should specify the name of your AWS keypair - as noted under AWS Specific Requirements above.

> **NOTE 2:** The above bind-mounts will map files and source directories to the correct locations within the control host container. Update the local paths per your environment for a successful run.

> **NOTE 3:** Alternatively to edit both `OSEv3.yml` and `all.yml` and set your AWS credentials there, you can define them in a file and pass it to the docker command.
```shell
docker run -u `id -u` \
      ...
      --env-file $HOME/aws-creds/my-aws-access-keys.src \
      ...
      redhatcop/installer-aws
```

Done! Wait till the provisioning completes and you should have an operational OpenShift cluster. If something fails along the way, either update your inventory and re-run the above `end-to-end.yml` playbook, or it may be better to [delete the cluster](https://github.com/redhat-cop/casl-ansible#deleting-a-cluster) and re-start.

## Updating a Cluster

Once provisioned, a cluster may be adjusted/reconfigured as needed by updating the inventory and re-running the `end-to-end.yml` playbook.

## Scaling Up and Down

A cluster's Infra and App nodes may be scaled up and down by editing the following parameters in the `all.yml` file and then re-running the `end-to-end.yml` playbook as shown above.

```yml
appnodes:
  count: <REPLACE WITH NUMBER OF INSTANCES TO CREATE>
infranodes:
  count: <REPLACE WITH NUMBER OF INSTANCES TO CREATE>
```

## Deleting a Cluster

A cluster can be decommissioned/deleted by re-using the same inventory with the `delete-cluster.yml` playbook found alongside the `end-to-end.yml` playbook.

```shell
docker run -it -u `id -u` \
      -v $HOME/.ssh/id_rsa:/opt/app-root/src/.ssh/id_rsa:Z \
      -v $HOME/src/:/tmp/src:Z \
      -e INVENTORY_DIR=/tmp/src/casl-ansible/inventory/sample.casl.example.com.d/inventory \
      -e PLAYBOOK_FILE=/tmp/src/casl-ansible/playbooks/openshift/delete-cluster.yml \
      -e OPTS="-e aws_key_name=my-key-name" -t \
      redhatcop/installer-aws
```

**TODO:** Using an existing VPC or maintaining the created VPC is not supported yet.
