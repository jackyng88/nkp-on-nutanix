# Simple NKP Installation Guide on a Nutanix Cluster 

- This is a simple installation guide for Nutanix Kubernetes Platform v2.14 on simple AHV VMs and usiing a Linux/MacOS machine.

## Prerequisites
- Download the Nutanix Kubernetes Platform (NKP) CLI/binary and Konvoy Image Builder (KIB) from Nutanix Portal [here](https://portal.nutanix.com/page/downloads?product=nkp). Choose your appropriate OS and the version.
- A container engine of some kind i.e. Docker or Podman. Docker Desktop is usually not supported in some enterprises so you may need to use Podman.
- A container registry for your environment
- A valid Nutanix account
- A properly configured Prism Central with administrator access with the following credentials
    - Managing the cluster i.e. listing subnets, creating VMs in Prism Central
    - Managing persistent storage used by the Nutanix Container Storage Interface (CSI) provider.
    - Discovering node metadata used by Nutanix Cloud Cost Management (CCM) provider.
- A subnet with unused IP addresses (this example has 9 unused)
    - One IP address for each node in the Kubernetes cluster. 
    - Default cluster size has *three* control plane nodes and *four* worker nodes for a total of *seven*
    - *One* IP address for the control plane endpoint Virtual IP
    - *One* IP address for the NKP LoadBalancer serice Virtual IP
- Since we will be standing up a self-managed NKP cluster i.e. a cluster that isn't for managing another cluster you will need to unset your ``KUBECONFIG`` environment variable and make sure ``~/.kube/config`` doesn't exist. Copy the current config file to somewhere else for use later.

## Setting up a Subnet in Prism Central
- In Prism Central, under Infrastructure at the top drop down, select ``Network & Security``
- Then click ``Subnets``
- Click ``Create Subnet``
- Fill out the following:
    - Enter a name for the Subnet under ``Name``
    - For ``Type`` select VLAN
    - Select the appropriate ``Cluster``
    - Under ``IP Address Management`` and then ``IP Assignemnt Service`` select ``Nutanix IPAM``
    - Under ``Network IP Address / Prefix`` make sure that you give it a proper CIDR notation network address.
    - Under ``IP Pools`` provide a range from ``Start Address`` to ``End Address``. Make sure this range falls under IP addresses available in the subnet.
    - Provide a ``Gateway IP Address`` that adheres to the previously given Network IP Address
    - Click ``Create``

## Creating a Storage Container in Prism Central
- We will need a Storage Container for usage with NKP and the necessary storage drivers provisioning storage.
- In Prism Central under ``Infrastructure``, select ``Storage`` and then ``Storage Containers``.
- Fill in the details and advanced configuration settings as needed. In this scenario, we will leave everything default.


## Configuring Prism Central Authorization Policy and Role
- In Prism Central, go to ``Admin Center`` and click ``IAM``
- Click ``Identities`` and then ``Add Local User``. Fill it out with the appropriate details.
- Now go to ``Authorization Policies`` and click ``Create Authorization Policy``. Fill out the Policy and assign it the proper ``Role`` with the necessary credentials outlined [here](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_14%3Atop-prism-central-role-permissions-r.html&a=9728ece46a91ccd8e14f9482b8693e927c28d3f08649b668180f06007cc059dbd062f5f7bdd998b8)


## Adding a Base Image for use with NKP
- In Prism Centraln we need to add an Image for use with the Nutanix Kubernetes Platform Image Builder (NIB) in order to create a custom image.
- In this example we will be assuming a NKP Starter License tier so we will need to use the pre-built image that was downloaded along with the NKP binary file.
- We will add a Rocky Linux 9.5 image that was downloaded from the portal [here](https://portal.nutanix.com/page/downloads?product=nkp&a=9728ece46a91ccd8e14f9482b8693e927c28d3f08649b668180f06007cc059dbd062f5f7bdd998b8)
- Under ``Infrastructure`` click the ``Compute`` drop-down and then ``Images``.
- Click ``Add Image`` and then add the Rocky Linux 9.5 image that was just downloaded.


## Creating a NKP Cluster
- This is going to perform a simple installation with a bit less customization option through the ``nkp`` cli tool.
- Export the following environment variables. Choose a cluster name of your choice and use your Prism Central credentials for `NUTANIX_USER` and `NUTANIX_PASSWORD`:
```sh
export CLUSTER_NAME=<name-of-nutanux-cluster>
export NUTANIX_USER=<username>
export NUTANIX_PASSWORD=<password>
```
- In your terminal/shell run the following command:
```sh
nkp create cluster nutanix
```
- Using the CLI tool fill out the following instructions, following the key commands to navigate the tool.
- Fields to fill:
    - ``Prism Central Endpoint``: Use your Prism Central endpoint with IP or FQDN appended with :9440
    - ``Username``: Your Prism Central username i.e. admin
    - ``Password``: Your Prism Central password for previous username
    - ``Insecure``: Select ``No`` if you require a certificate to connect to PC. ``Yes`` if you do not. 
    - ``Project``: The NKP CLI tool will fetch a project to choose from.
    - ``Prism Element Cluster``: Select from a fetched list of Prism Element clusters.
    - ``Subnet``: Select from a fetched list of subnets. Select the one we created earlier in the [Setting up a Subnet in Prism Central](#setting-up--subnet-in-prism-central) section.
    - ``Cluster Name``: If you exported the ``CLUSTER_NAME`` env variable earlier it will be populated here. Otherwise you can fill it in now.
    - ``Control Plane Endpoint``: Choose an unused IP address from your created ``Subnet``. If you do not know which ones are currently used go to Prism Central > ``Infrastructure`` > `` Network & Security`` > ``Subnets`` > ``Network Config`` button. Now next to your created subnet click on ``Used IP Addresses`` and you will see the currently used IP Addresses in this subnet. Pick an IP address not present here and one that does not fall under the IP Address Pool. 
    - ``VM Image``: A fetched list of *compatible* VM images should show up i.e. the Rocky Linux image we uploaded.

- Now under the ``Kubernetes Network`` section:
    - ``Service Load Balancer IP Range``: Supply a range for the Load Balancer IP in a range of the format ``x.x.x.x-y.y.y.y``. Make sure that this range falls under your subnet.
    - ``Pod Network``: Can be left as default i.e. ``192.168.0.0/16``. This is the IP internally to the Kubernetes cluster isolated by Pod. You can think of this as the Pod having it's own network and the internal containers within each Pod having each their own unique IP address. So container A within Pod A can have an IP address of say ``192.168.0.3`` as well as Container B within Pod B with the same IP Address.
    - ``Service Network``: Can be left as default i.e. ``10.96.0.0/12``. Created services within the Kubernetes service will have a VIP or ClusterIP in this address range.

- Under the ``Storage`` section:
    - ``Reclaim Policy``: The behavior in which when a user deletes a Persistent Volume Claim (PVC) whether the Kubernetes cluster will ``Retain`` or ``Delete`` the associated Persistent Volume (PV) dynamically created by a Storage Class (SC). If your application might have sensitive or important data you may use Retain. For this example we will use ``Delete``.
    - ``File System``:  For a simple usecase here, choose ``ext4`` for general use as XFS is typically better with large files.
    - ``Hypervisor Attached Volumes``: Select ``Yes``.
    - ``Storage Container``: Select the name of your created Storage Container from the section [Creating a Storage Container in Prism Central](#creating-a-storage-container-in-prism-central) that is fetched.

- We are now at the ``Additional Configuration (optional)`` section. Here you can fill in the details as necessary for use within the cluster, we will use the defaults in this scenario.

- Finally during the ``Create NKP Cluster?:`` prompt, we will select ``Create`` and hit Enter.

- We will not wait for the NKP CLI to create a bootstrap cluster which will then create the actual NKP cluster.







