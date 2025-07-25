cilium-kubernetes
=================

This Ansible role installs [Cilium](https://docs.cilium.io) network on a Kubernetes cluster. Behind the doors it uses the official [Helm chart](https://helm.cilium.io/). Currently procedures like installing, upgrading and deleting the Cilium deployment are supported.

Requirements
------------

You need to have [Helm 3](https://helm.sh/) binary installed on that host where `ansible-playbook` is executed or on that host where you delegated the playbooks to (e.g. by using `cilium_delegate_to` variable).

A properly configured `KUBECONFIG` is also needed (which is located in our repo's installer by default). Normally if `kubectl` works with your cluster then everything should be already fine.

Additionally the Ansible `kubernetes.core` collection needs to be installed. This can be done by using the `collections.yml` file included in this role: `ansible-galaxy install -r collections.yml`. Again, this is provided by the opus environment.

And of course you need a Kubernetes Cluster - But this is provided for you by Ubiquity ;-)

Role Variables
--------------

```
# Helm chart version
cilium_chart_version: "1.12.3"

# Helm chart name
cilium_chart_name: "cilium"

# Helm chart URL
cilium_chart_url: "https://helm.cilium.io/"

# Kubernetes namespace where Cilium resources should be installed
cilium_namespace: "cilium"

# etcd settings. If "cilium_etcd_enabled" variable is defined and set to "true",
# Cilium etcd settings are generated and deployed. Otherwise all the following
# "cilium_etcd_*" settings are ignored.
#
cilium_etcd_enabled: "true"

# Interface where etcd daemons are listening. If etcd daemons are bound to
# a WireGuard interface this setting should be "wg0" (by default) e.g.
cilium_etcd_interface: "eth0"

# Port where etcd daemons are listening
cilium_etcd_client_port: 2379

# Ansible etcd host group in Ansible's "hosts" file. This value is used in
# "templates/cilium_values_default.yml.j2" template to determine the IP
# addresses of the hosts where etcd daemons are listening.
cilium_etcd_nodes_group: "k8s_etcd"

# If this variable is defined a Kubernetes secret will be installed which
# contains the certificate files defined in "cilium_etcd_cafile",
# "cilium_etcd_certfile" and "cilium_etcd_keyfile"
#
# This causes that a secure connection (https) will be established to etcd.
# This of course requires that etcd is configured to use SSL/TLS.
#
# If this value is not defined (e.g. commented) the rest of the "cilium_etcd_*"
# settings below are ignored and connection to etcd will be established
# unsecured via "http".
cilium_etcd_secrets_name: "cilium-etcd-secrets"

# Where to find the certificate files for KCA defined below.
# You may already have "k8s_ca_conf_directory" variable defined which you
# can re-use here. This role also generates the certificate files that can
# be used for the variables below.
# By default this will be "$HOME/k8s/certs" of the current user that runs
# "ansible-playbook" command.
cilium_etcd_cert_directory: "{{ '~/k8s/certs' | expanduser }}"

# etcd certificate authority file (file will be fetched in "cilium_etcd_cert_directory")
cilium_etcd_cafile: "ca-etcd.pem"

# etcd certificate file (file will be fetched in "cilium_etcd_cert_directory")
# Make sure that the certificate contains the IP addresses in the "Subject
# Alternative Name" (SAN) of the interfaces where etcd daemons listens on
# (that's the IP addresses of the interfaces defined in "cilium_etcd_interface").
cilium_etcd_certfile: "cert-cilium.pem"

# etcd certificate key file (file will be fetched in "cilium_etcd_cert_directory")
cilium_etcd_keyfile: "cert-cilium-key.pem"

# By default all tasks that needs to communicate with the Kubernetes
# cluster are executed on your local host (127.0.0.1). But if that one
# doesn't have direct connection to this cluster or should be executed
# elsewhere this variable can be changed accordingly. 
cilium_delegate_to: 127.0.0.1

# Shows the "helm" command that was executed if a task uses Helm to
# install, update/upgrade or deletes such a resource.
cilium_helm_show_commands: false

# Without "action" variable defined this role will only render a file
# with all the resources that will be installed or upgraded. The rendered
# file with the resources will be called "template.yml" and will be
# placed in the directory specified below.
cilium_template_output_directory: "{{ '~/cilium/template' | expanduser }}"
```

Usage:
------

The first thing to do is to check `templates/cilium_values_default.yml.j2`. This file contains the values/settings for the Cilium Helm chart that are different to the default ones which are located [here](https://github.com/cilium/cilium/blob/master/install/kubernetes/cilium/values.yaml). The default values of this Ansible role are using a TLS enabled `etcd` cluster. If you have a self hosted/bare metal Kubernetes cluster chances are high that there is already running an `etcd` cluster for the Kubernetes API server which is the case for me.

The `templates/cilium_values_default.yml.j2` template also contains some `if` clauses to use an `etcd` cluster that is not TLS enabled. See `defaults/main.yml` to check which values can be changed. You can also introduce your own variables and use it in `templates/cilium_values_user.yml.j2` if you want of course.

But nothing is made in stone ;-) To use your own values just create a file called `cilium_values_user.yml.j2` and put it into the `templates` directory. Then this Cilium role will use that file to render the Helm values. You can use `templates/cilium_values_default.yml.j2` as a template or just start from scratch. As mentioned above you can modify all settings for the Cilium Helm chart that are different to the default ones which are located [here](https://github.com/cilium/cilium/blob/master/install/kubernetes/cilium/values.yaml).

After the values file (`templates/cilium_values_default.yml.j2` or `templates/cilium_values_user.yml.j2`) is in place and the `defaults/main.yml` values are checked the role can be installed. Most of the role's tasks are executed locally by default so to say as quite a few tasks need to communicate with the Kubernetes API server or executing [Helm](https://helm.sh/) commands. But you can delegate this kind of tasks to a different host by using `cilium_delegate_to` variable (see above).

```
ansible-playbook --tags=role-cilium-kubernetes k8s.yml
```

To render the template into a different directory use `cilium_template_output_directory` variable e.g.:

```bash
ansible-playbook --tags=role-cilium-kubernetes --extra-vars cilium_template_output_directory="/tmp/cilium" k8s.yml
```

If you want to see the `helm` commands and the parameters which were executed in the logs you can also specify `--extra-vars cilium_helm_show_commands=true`.

One of the final tasks is called `TASK [githubixx.cilium_kubernetes : Write templates to file]`. This renders the template with the resources that will be created into the directory specified in `cilium_template_output_directory`. The file will be called `template.yml`. The directory/file will be placed on your local machine to be able to inspect it.

If the rendered output contains everything you need the role can be installed which finally deploys Cilium:

```
ansible-playbook --tags=role-cilium-kubernetes --extra-vars action=install k8s.yml
```

To check if everything was deployed use the usual `kubectl` commands like `kubectl -n <cilium_namespace> get pods -o wide`.

As [Cilium](https://docs.cilium.io) issues updates/upgrades every few weeks/months the role also can do upgrades. The role basically executes what is described in [Cilium upgrade guide](https://docs.cilium.io/en/v1.12/operations/upgrade/). That means the Cilium pre-flight check will be installed and some checks are executed before the update actually takes place. Have a look at `tasks/upgrade.yml` to see what's happening before, during and after the update. Of course you should consult [Cilium upgrade guide](https://docs.cilium.io/en/v1.12/operations/upgrade/) in general to check for major changes and stuff like that before upgrading. [Roll back](https://docs.cilium.io/en/v1.12/operations/upgrade/#step-3-rolling-back) is currently not implemented but can be easily done via `kubectl` as described in the upgrade document.

Before doing the upgrade you basically only need to change `cilium_chart_version` variable e.g. from `1.10.10` to `1.11.4` to upgrade from `1.10.10` to `1.11.4`. So to do the update run

```
ansible-playbook --tags=role-cilium-kubernetes --extra-vars action=upgrade k8s.yml
```

As already mentioned the role already includes some checks to make sure the upgrade runs smooth but you should again check with `kubectl` if all works as expected after the upgrade.

And finally if you want to get rid of Cilium you can delete all resources again:

```
ansible-playbook --tags=role-cilium-kubernetes --extra-vars action=delete k8s.yml
```

If you don't have any CNI plugins configured this will cause `kubelet` process on the Kubernetes worker nodes to issue CNI errors every now and then because there is no CNI related stuff anymore and of course connectivity between pods on different hosts will be gone together with any network policies and stuff like that.

Example Playbook
----------------

Example 1 (without role tag):
```
- hosts: k8s_worker
  roles:
    - cilium-kubernetes
```

Example 2 (assign tag to role):
```
-
  hosts: k8s_worker
  roles:
    -
      role: cilium-kubernetes
      tags: role-cilium-kubernetes
```

