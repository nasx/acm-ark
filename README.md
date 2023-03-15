# Generating kubeconfigs for oc/kubectl Access to Managed Clusters Proxied through ACM Hub

Use case: I have an OpenShift cluster managed by ACM but have no direct access to the managed cluster API. I want to run oc/kubectl commands against the managed cluster.

Want to start hacking? Jump strait to [usage](#usage).

## Background

Advanced Cluster Management (ACM) ships with a `cluster-proxy-addon` that uses a reverse proxy server ([ANP](https://github.com/kubernetes-sigs/apiserver-network-proxy)) to send API requests from the hub to a managed cluster. This is particularly useful when the managed cluster API is behind a firewall or is otherwise inaccessible. Using this addon, an administrator can also build a kubeconfig file to execute remote commands against a managed cluster using oc/kubectl proxied through ACM. The [documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/clusters/index#cluster-proxy-addon) to do this is relatively simple, but it requires a lot of text parsing and it assumes a user has access API access to both the managed cluster and the hub to do the initial configuration.

This small project (ACM ARK, or Automatic Remote Kubeconfig) was designed to simplify the process of generating a 'proxied' kubeconfig outlined in the ACM documentation. It has two objectives:

* Automate the creation of your kubeconfig with Ansible (that's why we are here, right?)
* More importantly, assume you have no connectivity to the managed cluster for initial configuration (i.e. orchestrate everything through the hub cluster)

## A Brief Overview of the Orchestration

The first step of the documentation is to create a service account on the managed cluster and grab the authentication token to embed in our 'proxied' kubeconfig file. To automate this process, we are going to use ACM's Managed Service Account Addon (in tech preview as of 2.7). This gives us a couple of cool capabilities:

* Lifecycle of service account on managed cluster can be handled by the hub
* Managed cluster reports service account token back to hub, which is stored as a secret in the managed cluster's namespace
* `ManagedServiceAccount` resource has conditions Ansible can verify

Next, we need to apply some sort of RBAC to the service account. This is done via an ACM policy, which creates a `ClusterRoleBinding` for the service account on the managed cluster. The policy is configured with `pruneObjectBehavior: DeleteAll`, so when deleted the `ClusterRoleBinding` will be deleted as well.

Finally, use a Jinja template to generate our 'proxied' kubeconfig. The following parameters are templatized:

|Parameter|Source|
|:---|:---|
|certificate-authority-data|Taken from `kube-root-ca.crt` `ConfigMap` in the `openshift-service-ca` `Namespace` on the hub|
|server|The user server (an HTTP proxy server that receives HTTP requests from users and forwards them to the ANP proxy server) is exposed as a `Route`. To access a specific managed cluster, the name is appended to the end of the url (i.e. `https://<route-spec-host>/<managed-cluster-name>`)|
|context|Several variables that represent the name of the managed cluster and the name of the service account used to access the managed cluster|
|token|Taken from `Secret` generated by `ManagedServiceAccount` resource on the hub|

## Usage

First let's clone this repository. Locally, find a place to store it and run the following:

```shell
git clone https://github.com/nasx/acm-ark.git
```

Next, make sure your KUBECONFIG is set to the ACM hub cluster.

```shell
export KUBECONFIG=/path/to/hub/kubeconfig
```

Before we go further, lets review the requirements.

### ACM Requirements

The Managed Service Account Addon needs to be enabled on the hub cluster (if this is already enabled on your hub, skip to the [Ansible Requirements](#ansible-requirements)). To do this, we can patch the `MultiClusterHub` resource as follows:

```shell
export MULTICLUSTERHUB=$(oc get multiclusterhub -n open-cluster-management -ojsonpath='{.items[].metadata.name}')
```

Verify MULTICLUSTERHUB is correct:

```shell
echo $MULTICLUSTERHUB
```

Patch the MultiClusterHub resource to enable the addon:

```shell
oc patch multiclusterhub $MULTICLUSTERHUB -n open-cluster-management --type=json -p='[{"op": "add", "path": "/spec/overrides/components/-","value":{"name":"managedserviceaccount-preview","enabled":true}}]'
```

### Ansible Requirements

Ansible has some Python dependencies, so let's create a virtual environment with everything we need.

To make things easy, we can just create it locally in our folder with the git repository:

```shell
mkdir ./local
```

```shell
python -m venv local/acm-ark-venv
```

Now that we have our virtual environment, let's use it by activating it:

```shell
source local/acm-ark-venv/bin/activate
```

If everything goes well, your prompt should be updated with the name of the virtual environment.

Finally, lets install Ansible and add our Python dependencies.

```shell
python -m pip install -U pip
```

```
python -m pip install -U ansible openshift jmespath
```

### Install Ansible Collections

Our playbooks rely on a couple of collections. In the git repo, switch to the `ansible` directory. Install them as follows:

```shell
ansible-galaxy collection install -r collections/requirements.yaml
```

### Setting up vars.yaml

The following variables are included in `vars.yaml`:

|Variable|Description|
|:---|:---|
|ark_service_account_name|The name of the ManagedServiceAccount resource on the hub and ServiceAccount resource on the managed cluster|
|managed_cluster_deploy_msa_addon|If set to `true`, enable the Managed Service Account Addon on the managed cluster|
|managed_cluster_name|The name of the managed cluster we are trying to access|
|msa_remote_cluster_role|The name of the `ClusterRole` we want to bind to our service account on the managed cluster (the role must already exist)|

By default, the 'proxied' kubeconfig is saved in `/tmp/<managed_cluster_name>-ark-kubeconfig`. This can be customized by including the following variables in `vars.yaml`:

```yaml
remote_kubeconfig_directory: /path/to/directory
remote_kubeconfig_file: proxied-kubeconfig
```

### Run the Playbook

Once you are happy with `vars.yaml`, we can run the `ark.yaml` playbook as follows:

```shell
ansible-playbook -e @vars.yaml playbooks/ark.yaml
```

### Accessing Remote Cluster

Once the playbook finishes, you should have your shiny new 'proxied' kubeconfig. Export it and try it out:

```shell
$ oc whoami
system:serviceaccount:open-cluster-management-agent-addon:acm-ark
```
