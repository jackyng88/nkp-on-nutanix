# Simple NKP Installation Guide on a Nutanix Cluster

- This is a simple installation guide for Nutanix Kubernetes Platform v2.17 on simple AHV VMs and using a Linux/MacOS machine.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setting up a Subnet in Prism Central](#setting-up-a-subnet-in-prism-central)
- [Creating a Storage Container in Prism Central](#creating-a-storage-container-in-prism-central)
- [Configuring Prism Central Authorization Policy and Role](#configuring-prism-central-authorization-policy-and-role)
- [Adding a Base Image for use with NKP](#adding-a-base-image-for-use-with-nkp)
- [Deploying an NKP Cluster](#deploying-an-nkp-cluster)
- [(optional) Adding Harbor Container Registry to the Cluster](#optional-adding-harbor-container-registry-to-the-cluster)
    - [Deploying CloudNativePG](#deploying-cloudnativepg)
    - [Adding Nutanix Object Store](#adding-nutanix-object-store)

## Prerequisites
- Download the Nutanix Kubernetes Platform (NKP) CLI/binary and Konvoy Image Builder (KIB) from Nutanix Portal [here](https://portal.nutanix.com/page/downloads?product=nkp). Choose your appropriate OS and the version. **Note**: Make sure that the appropriate Linux OS image maps to the NKP CLI version.
- A container engine of some kind i.e. Docker or Podman. Docker Desktop is usually not supported in some enterprises so you may need to use Podman, containerd, etc. In my case I used Rancher Desktop with `QEMU` emulation and `dockerd` for the container runtime engine on a arm64 MacOS machine.
  - The container runtime engine (CRE) will allow your laptop to serve as a bastion host to deploy a KinD (Kubernetes in Docker) cluster to serve as the bootstrap cluster.
  - For ease of use depending on your environment - add the nkp/docker (or related CLI binaries) to your PATH environment variable instead of having to use the entire working directory to those binaries.
- (Optional) A container registry for your environment 
  - For air-gapped deployments your bastion host will need to push the air gapped bundles into an external container registry like Harbor.
- A valid Nutanix account
- A properly configured Prism Central with administrator access with the following credentials
    - Managing the cluster i.e. listing subnets, creating VMs in Prism Central
    - Managing persistent storage used by the Nutanix Container Storage Interface (CSI) provider.
    - Discovering node metadata used by Nutanix Cloud Cost Management (CCM) provider.
- A subnet with unused IP addresses (this deployment example has 9 unused)
    - One IP address for each node in the Kubernetes cluster. 
      - Default cluster size has *three* control plane nodes and *four* worker nodes for a total of *seven* IP addresses.
    - *One* IP address for the control plane endpoint Virtual IP
    - *One* IP address for the NKP LoadBalancer service Virtual IP for MetalLB.
- Since we will be standing up a self-managed NKP cluster i.e. a cluster that isn't for managing another cluster you will need to unset your ``KUBECONFIG`` environment variable and make sure ``~/.kube/config`` doesn't exist. Copy the current config file to somewhere else for use later.
  - This is for purposes of administrating your Kubernetes cluster via `kubectl`. By default without providing the `--kubeconfig=/path/to/kubeconfig`, any `kubectl` command will look to the `KUBECONFIG` environment variable to be set to the path.

## Setting up a Subnet in Prism Central
- *Note*: Ideally you have a subnet with DHCP/Nutanix IPAM enabled with a IP pool. These are used to provision NKP Node VMs. 
  - You can deploy baremetal/pre-provisioned VMs using static IPs but typically that is not advised.
- In Prism Central, under Infrastructure at the top drop down, select ``Network & Security``
- Then click ``Subnets``
- Click ``Create Subnet``
- Fill out the following:
    - Enter a name for the Subnet under ``Name``
    - For ``Type`` select VLAN
    - Select the appropriate ``Cluster``
    - Under ``IP Address Management`` and then ``IP Assignment Service`` select ``Nutanix IPAM``
    - Under ``Network IP Address / Prefix`` make sure that you give it a proper CIDR notation network address.
    - Under ``IP Pools`` provide a range from ``Start Address`` to ``End Address``. Make sure this range falls under IP addresses available in the subnet.
    - Provide a ``Gateway IP Address`` that adheres to the previously given Network IP Address
    - Click ``Create``

## Creating a Storage Container in Prism Central
- We will need a Storage Container for usage with NKP and the necessary storage drivers provisioning storage.
- In Prism Central under ``Infrastructure``, select ``Storage`` and then ``Storage Containers``.
- For capacity of the Storage container - it depends on your workload needs but typically ~80 gb for each node and around ~120 gb for the NKP services.
- Fill in the details and advanced configuration settings as needed. In this scenario, we will leave everything default.


## Configuring Prism Central Authorization Policy and Role
- In Prism Central, go to ``Admin Center`` and click ``IAM``
- Click ``Identities`` and then ``Add Local User``. Fill it out with the appropriate details.
- Now go to ``Authorization Policies`` and click ``Create Authorization Policy``. Fill out the Policy and assign it the proper ``Role`` with the necessary credentials outlined [here](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_17%3Atop-prism-central-role-permissions-r.html)


## Adding a Base Image for use with NKP
- In Prism Central we will add the necessary OS image.
- In this example we will be assuming a NKP Starter License tier so we will need to use the pre-built image that was downloaded along with the NKP binary file.
  - In higher NKP tiers (Pro, Ultimate) you will be able to use other OS images as well as custom KIB-built images.
- We will add a Rocky Linux 9.6 image that was downloaded from the portal [here](https://portal.nutanix.com/page/downloads?product=nkp)
- Under ``Infrastructure`` click the ``Compute`` drop-down and then ``Images``.
- Click ``Add Image`` and then add the Rocky Linux 9.6 image that was just downloaded.
![NKP Image Select](./images/nkp-image-select.png)

- Choose a location for your image, for simplicity choose ``Place image directly on clusters``
![NKP Image Location](./images/nkp-image-location.png)



## Deploying an NKP Cluster
- This is going to perform a simple installation with a bit less customization option through the ``nkp`` cli tool. This utilize an interactive shell window.
- To further operationalize this deployment with the `nkp` cli you will need to utilize flags for the `nkp` command - especially in tandem with scripts or GitOps configurations - but for the purposes of a quick lab/demo the shell command is fine. It will default a 3 control plane/4 worker plane cluster.
- The following is an example shell command with more flag options:
```sh

nkp create cluster nutanix \
  --cluster-name                              "${CLUSTER_NAME}" \
  --kubernetes-version                        "${KUBERNETES_VERSION}" \
  \
  # ── Prism Central endpoint ──────────────────────────────────────────
  --endpoint                                  "${NUTANIX_PC_ENDPOINT}:9440" \
  --insecure                                  false \
  \
  # ── CAPX credentials (VM lifecycle — stored in capx-system Secret) ──
  --nutanix-username                          "${CAPX_USERNAME}" \
  --nutanix-password                          "${CAPX_PASSWORD}" \
  \
  # ── Infrastructure placement ────────────────────────────────────────
  --control-plane-prism-element-cluster       "${NUTANIX_PE_CLUSTER}" \
  --control-plane-subnets                     "${NUTANIX_SUBNET}" \
  --control-plane-vm-image                    "${NODE_IMAGE_NAME}" \
  --control-plane-vcpus                       4 \
  --control-plane-cores-per-vcpu              1 \
  --control-plane-memory-gib                  16 \
  --control-plane-disk-gib                    100 \
  --control-plane-replicas                    3 \
  --control-plane-endpoint-ip                 "${CONTROL_PLANE_VIP}" \
  \
  --worker-prism-element-cluster              "${NUTANIX_PE_CLUSTER}" \
  --worker-subnets                            "${NUTANIX_SUBNET}" \
  --worker-vm-image                           "${NODE_IMAGE_NAME}" \
  --worker-vcpus                              8 \
  --worker-cores-per-vcpu                     1 \
  --worker-memory-gib                         32 \
  --worker-disk-gib                           100 \
  --worker-replicas                           4 \
  \
  # ── CSI credentials (PV lifecycle — stored in kube-system Secret) ───
  --csi-username                              "${CSI_USERNAME}" \
  --csi-password                              "${CSI_PASSWORD}" \
  --csi-storage-container                     "${NUTANIX_STORAGE_CONTAINER}" \
  --csi-reclaim-policy                        "Retain" \
  \
  # ── Networking ──────────────────────────────────────────────────────
  --kubernetes-service-load-balancer-ip-range "${LB_IP_RANGE}" \
  --cni                                       "calico" \
  \
  # ── SSH access ──────────────────────────────────────────────────────
  --ssh-public-key-file                       "${SSH_PUBLIC_KEY_FILE}" \
  \
  # ── Target the management cluster (post-pivot) ──────────────────────
  --self-managed \
  --kubeconfig                                "${KUBECONFIG}"
```
- For an exhaustive list on flags visit [here](https://docs.d2iq.com/dkp/2.8/nkp-create-cluster-nutanix) or by running `nkp create cluster nutanix --help`.
- In your terminal/shell run the following command:
```sh
nkp create cluster nutanix
```
- The interactive CLI tool will look like the following:
![NKP Interactive](./images/nkp-interactive.png)


- Using the CLI tool fill out the following instructions, following the key commands to navigate the tool.
- Fields to fill:
    - ``Prism Central Endpoint``: Use your Prism Central endpoint with IP or FQDN appended with :9440
    - ``Username``: Your Prism Central username i.e. admin
    - ``Password``: Your Prism Central password for previous username
    - ``Insecure``: Select ``No`` if you require a certificate to connect to PC. ``Yes`` if you do not. 
    - ``Project``: The NKP CLI tool will fetch a project to choose from.
    - ``Prism Element Cluster``: Select from a fetched list of Prism Element clusters.
    - ``Subnet``: Select from a fetched list of subnets. Select the one we created earlier in the [Setting up a Subnet in Prism Central](#setting-up-a-subnet-in-prism-central) section.
    - ``Cluster Name``: Fill it in with a name of your choosing.
    - ``Control Plane Endpoint``: Choose an unused IP address from your ``Subnet``. If you do not know which ones are currently used go to Prism Element > Settings Icon in the top right > `` Network Configuration`` > `Unused IP Addresses` link. Now next to your created subnet click on ``Used IP Addresses`` and you will see the currently used IP Addresses in this subnet. 
      - **NOTE**: Pick an IP address currently not used and one that does not fall under the DHCP/IPAM range as well in the ``Service Load Balancer IP Range`` a bit further.
    - ``VM Image``: A fetched list of *compatible* VM images should show up i.e. the Rocky Linux image we uploaded.

- Now under the ``Kubernetes Network`` section:
    - ``Service Load Balancer IP Range``: Supply a range for the Load Balancer IP in a range of the format ``x.x.x.x-y.y.y.y``. 
      - **Note**: Similar to the control plane endpoint VIP, make sure that this range falls under your subnet and outside of your DHCP IP pool.
    - ``Pod Network``: Can be left as default i.e. ``192.168.0.0/16``. This is the IP internally to the Kubernetes cluster isolated by Pod. You can think of this as the Pod having its own network and the internal containers within each Pod having each their own unique IP address. So container A within Pod A can have an IP address of say ``192.168.0.3`` as well as Container B within Pod B with the same IP Address.
    - ``Service Network``: Can be left as default i.e. ``10.96.0.0/12``. Created services within the Kubernetes service will have a VIP or ClusterIP in this address range.

- Under the ``Storage`` section:
    - ``Reclaim Policy``: The behavior in which when a user deletes a Persistent Volume Claim (PVC) whether the Kubernetes cluster will ``Retain`` or ``Delete`` the associated Nutanix vDisk in the event of a PVC deletion. If your application might have sensitive or important data you may use Retain. For this example we will use ``Delete``.
      - Generally you would use `Delete` on workload clusters and `Retain` on management clusters where you have platform data (Prometheus, Loki, Grafana, etc.)
    - ``File System``:  For a simple use case here, choose ``ext4`` for general use as XFS is typically better with large files and when you need high-throughput sequential I/O i.e. Kafka.
    - ``Hypervisor Attached Volumes``: Select ``Yes``.
    - ``Storage Container``: Select the name of your created Storage Container from the section [Creating a Storage Container in Prism Central](#creating-a-storage-container-in-prism-central) that is fetched.

- We are now at the ``Additional Configuration (optional)`` section. Here you can fill in the details as necessary for use within the cluster, we will mostly use defaults here besides the following:
    - The kubelet on the Kubernetes cluster may fail to pull images due to an un-authenticated pull rate limit. To overcome that you should provide DockerHub credentials. A personal/free account should be enough.
    - Under the ``Registry`` section:
        - For ``Registry URL`` use ``https://registry-1.docker.io``
        - Leave ``Registry CA Certificate`` blank
        - For ``Registry Username`` type in your DockerHub username
        - For ``Registry Password`` you can either use your DockerHub password or a generated personal access token which is advised.

- Finally during the ``Create NKP Cluster?:`` prompt, we will select ``Create`` and hit Enter. The final config will look similar to the following image:

![NKP Final Config](./images/nkp-final-config.png)

- We will now wait for the NKP CLI to create a bootstrap cluster which will then create the actual NKP cluster.
    - If you want to follow progress of creation of some resources via Prism Central UI you will initially see one VM for the bootstrap node and then 7 nodes in total for the self-managed standup (3 control plane nodes and 4 worker nodes).

    ![nodes](./images/nodes.png)

- Eventually when the cluster infrastructure, control-plane, machines, CAPI components, ClusterClass resources are ready, NKP will destroy the bootstrap cluster and create a kubeconfig file. 

![New Kubeconfig file](./images/kubeconfig-file.png)

- When the cluster successfully deploys you should see the following messages.
![NKP Successful Deployment](./images/nkp-successful-deploy.png)

- You may need to wait for Kommander to fully be ready and you can watch that with the following command:
```sh
kubectl -n kommander wait --for condition=Ready helmreleases --all --timeout 15m
```

- The logs will look like the following when ready:
```
helmrelease.helm.toolkit.fluxcd.io/cluster-observer-2360587938 condition met
helmrelease.helm.toolkit.fluxcd.io/dex condition met
helmrelease.helm.toolkit.fluxcd.io/dex-k8s-authenticator condition met
helmrelease.helm.toolkit.fluxcd.io/gatekeeper condition met
helmrelease.helm.toolkit.fluxcd.io/gatekeeper-proxy-mutations condition met
helmrelease.helm.toolkit.fluxcd.io/gateway-api-crds condition met
helmrelease.helm.toolkit.fluxcd.io/karma-traefik-certs condition met
helmrelease.helm.toolkit.fluxcd.io/kommander condition met
helmrelease.helm.toolkit.fluxcd.io/kommander-appmanagement condition met
helmrelease.helm.toolkit.fluxcd.io/kommander-operator condition met
helmrelease.helm.toolkit.fluxcd.io/kommander-ui condition met
helmrelease.helm.toolkit.fluxcd.io/kube-oidc-proxy condition met
helmrelease.helm.toolkit.fluxcd.io/kubefed condition met
helmrelease.helm.toolkit.fluxcd.io/prometheus-traefik-certs condition met
helmrelease.helm.toolkit.fluxcd.io/reloader condition met
helmrelease.helm.toolkit.fluxcd.io/traefik condition met
helmrelease.helm.toolkit.fluxcd.io/traefik-crds condition met
helmrelease.helm.toolkit.fluxcd.io/traefik-forward-auth-mgmt condition met
```

- You can run a few ``kubectl`` commands to test connectivity such as:
```sh
kubectl get pods
kubectl get nodes
kubectl get kommander -n kommander
```
- If the above commands don't work and you get a connectivity/API server connection error you will need to set your ``KUBECONFIG`` env variable to the newly created ``$CLUSTER_NAME-cluster.conf`` file in the current working directory.
```sh
export KUBECONFIG=$CLUSTER_NAME-cluster.conf
```

- When the deployment finishes, unless you specified an `--output-directory` flag, the KUBECONFIG file will be generated in the current working directory.
- If you need to regenerate the KUBECONFIG file you can run the following command.`Note`: the `$CLUSTER_NAME` is the name you provided earlier during the deployment:
```sh
kubectl get secret ${CLUSTER_NAME}-kubeconfig -n default -o jsonpath='{.data.value}' | base64 -d > $CLUSTER_NAME.conf
```

- Now to test the web GUI url use the following command:
```sh
nkp open dashboard --kubeconfig=${CLUSTER_NAME}.conf
```
- Once that command is entered, the login page should automatically open in your browser with the following credentials displayed in shell:
```sh
Username: <generated-username>
Password: <generated-password>
URL: https://<ip-address>/dkp/kommander/dashboard
```

![Dex Login Page](./images/dex-login.png)

- To find the credentials again if you happen to lose them, run the following command. The initial credentials live in a secret `dkp-credentials` in the `kommander` namespace:

```sh
kubectl get secret dkp-credentials -n kommander -o go-template='Username: {{.data.username|base64decode}}{{"\n"}}Password: {{.data.password|base64decode}}{{"\n"}}'
```

## (Optional) Adding Harbor Container Registry to the Cluster
- By default NKP clusters do not have their own dedicated internal private registry. NKP with 2.17 now supports Harbor for their internal registry. The following are steps to deploy it into your NKP cluster.
- Prerequisites:
    - Deploy the Cloud Native Postgres (CloudNativePG) operator and then enabling the CloudNativePG application on the cluster. We will be deploying the CNPG operator through ``Helm`` as it is a good tool to have and use.
    - Having access to an S3-compatible object store such as ``Nutanix Objects``, ``AWS S3``, or an integrated ``Rook Ceph``. For this scenario we will be using ``Nutanix Objects``.

### Deploying CloudNativePG
- We will now deploy the CNPG operator via Helm chart. The Helm charts are [here](https://github.com/cloudnative-pg/charts). Run the following command to add the Helm chart into your internal repo:
```sh
helm repo add cnpg https://cloudnative-pg.github.io/charts
```

- The following command will install the operator:
```sh
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
```

- You will see the following:
```
Release "cnpg" does not exist. Installing it now.
NAME: cnpg
LAST DEPLOYED: Fri Apr 11 11:13:51 2025
NAMESPACE: cnpg-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CloudNativePG operator should be installed in namespace "cnpg-system".
You can now create a PostgreSQL cluster with 3 nodes as follows:

cat <<EOF | kubectl apply -f -
# Example of PostgreSQL cluster
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example

spec:
  instances: 3
  storage:
    size: 1Gi
EOF

kubectl get -A cluster
```

- You can verify the resources that get created with the operator in the ``cnpg-system`` namespace via:
```sh
kubectl get all -n cnpg-system
```

- Next we'll deploy an actual CNPG database cluster:
```sh
helm upgrade --install cnpg-database \
  --namespace database \
  --create-namespace \
  cnpg/cluster
```

- You will then see the following:
![CNPG Installation](./images/cnpg-install.png)

- You can see the created resources in the ``database`` namespace via:
```sh
kubectl get all -n database
```

- It will take awhile for everything to spin up and reconcile but you can watch the progression of the CNPG database cluster pods with the following command:
```sh
kubectl get pods -n database -w
```

- When everything is finished you should see three `Running` pods:
```sh
kubectl get pods -n database
```
- The output:
```sh
NAME                      READY   STATUS    RESTARTS   AGE
cnpg-database-cluster-1   1/1     Running   0          4m22s
cnpg-database-cluster-2   1/1     Running   0          2m31s
cnpg-database-cluster-3   1/1     Running   0          65s
```

- For a sanity test on the database pods, exec into the exposed service:
```sh
kubectl -n database exec --stdin --tty services/cnpg-database-cluster-rw -- bash
```

- Within `bash` on the container run `psql`:
```sh
postgres@cnpg-database-cluster-1:/$ psql
psql (16.8 (Debian 16.8-1.pgdg110+1))
Type "help" for help.
```

- And you can then test with a command like the following to see system tables:
```psql
\dt *.*
```

- Now quit the psql CLI
```
\q
```

- Exit the exec
```sh
exit
```

### Adding Nutanix Object Store
- Navigate to `Prism Central` and in the top-left drop-down menu select `Objects` under `Unified Storage`
![Prism Central Objects](./images/pc-objects.png)

- Click the `Create Object Store` button and then `Confirm` on the `Create Object Stores: Prerequisites` window.

- In the following window supply the following details:
    - `Object Store Name`: Give it a name that is easily identifiable
    - `Domain`: It should automatically pull the domain name
    - `Cluster`: Select the cluster to use
    - `Worker Nodes`: By default, it's set to 3 but for this scenario we'll use `2` Worker Nodes.
- Click `Next` and configure the following:
    - `Storage Network`: Select your subnet where your CVMs ideally use IP addresses from
    - `Object Store Internal IPs`: Provide two IP addresses that exist in the same subnet as your CVM and not being currently used
    - `Public Network`: Select your subnet, can be similar to the one used for NKP cluster creation
    - `Public Network Static IPs`: Select an IP address outside of the IPAM pool and unused.

- The following is an example of final configuration:

![Objects Final Config](./images/objects-final-config.png)


- Wait for the Object Store validation checks to all pass. You will be unable to create your Object Store until that happens. Hit `Create Object Store` when the checks have all been validated.

- Wait for `Nutanix Objects` to finish creating your Object store
![Nutanix Objects Creating](./images/objects-creating.png)


- In `Objects` go to `Access Keys` and click `Add People` to generate access keys for API usage.
- Search for people or Add people not in a directory service. Lastly, generate the keys.
- Click `Download Key` and save it to your local machine. The txt file will have the `Secret Key`. You will only be able to see the `Access Key` within the Objects UI. We will need both keys later for the `COSI Driver for Nutanix` application.




### Enabling CloudNativePG with your Cluster
- Ensure that you have NKP Ultimate license. If you do not go to the Nutanix Portal and request a NKP Ultimate license key and apply it to your NKP cluster.

- In your NKP Web console in the top left click the drop down menu that says Global and select `Management Cluster Workspace`
![NKP Management Cluster Workspace](./images/nkp-mgmt-ws.png)

- On the left side of the menu click ``Applications``
- Then filter the name for `CloudNativePG`.
- Click the three dots on the `CloudNativePG` tile and click `Enable` and then `Enable` again.
![CloudNativePG Tile](./images/cnpg-tile.png)

- Next go back to the `Application Dashboard` and filter name by `COSI Driver` and click `Enable`
![COSI Driver NTNX](./images/cosi-tile.png)

- In `Enable Workspace Platform Application` box, fill out the following details:
    - `Prism Central Endpoint`: Your Prism Central URL likely in the form of `https://<ip>:9440`
    - `Prism Central Username`: Your Prism Central administrator username i.e. `admin`
    - `Prism Central Password`: Your Prism Central administrator password
    - `Nutanix Object Storage Endpoint`: The Public IP of your Object Store.
    - `Nutanix Object Access Key`: Generated in the previous section
    - `Nutanix Object Secret Key`: Generated in the previous section. In the downloaded `.txt` file.
![COSI Driver](./images/cosi-driver.png)

- Click `Enable`


- Again, go back to the `Application Dashboard` and filter name by `Harbor`. Do the same as CNPG and Enable it.
![Harbor Tile](./images/harbor-tile.png)

- Under `S3 Object Store` click the drop-down and select `Nutanix Objects`. If you did not properly setup the `COSI Driver for Nutanix` application, this option will be greyed out.
![Harbor Objects](./images/harbor-objects.png)

- Click `Enable` to enable `Harbor`.

- **NOTE**: You may need to enable more resources for your compute/worker nodes. You can do so by going to `Management Cluster` under your `Management Cluster Workspace` and then clicking `Nodepools`. 

- Click the three dots for the `Worker` node with a `Node Count` of 4.
![Edit Worker Node Resources](./images/worker-node-resources.png)

- Bump up the resources as you see fit. In this example, I used 16 `vCPUs` and 48gb `Memory`.
![Worker Node Edit](./images/worker-node-edit.png)
- Click `Save` and wait for Prism Central to finish provisioning. Prism Central will provision new VMs one by one and then condemn the old VMs. Some functions may go down/not be accessible intermittently in the NKP cluster.

- Run the following to find the address of the Harbor container registry:
```sh
echo "https://$(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.address}'):5000"
```

- Now to grab the password of Harbor:
```sh
kubectl get secrets -n ncr-system harbor-admin-password -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d
```

- What the above command does is data is converted to base64 encoding when placed in the `data` field of a Kubernetes secret. It will be plain text if placed in `stringData`. The output will look something like: 
```
?YmxhaCBibGFoCg==
```

- Enter the Harbor Registry URL and input the following values:
    - `Username`: `admin`
    - `Password`: The output of the base64 decoded password above.

![Harbor Login](./images/harbor-login.png)



## Using Harbor as the Cluster's Pull Through Cache
- **Context:** Typically without any configuration Harbor will just act as an image/object registry for you to push/pull from manually. We're going to configure Harbor to be used as a pull through cache. What this means is that Harbor will serve as a proxy and save images internally as well as be used for internal Kubernetes image builds. For example, Harbor will proxy an image pull request to Docker Hub, store it in its registry and then when you spin up a Pod or Deployment, the image will first be pulled from Harbor instead.
- First, we will create a custom user in Harbor. Typically these can be users like machine accounts (just for image pulling) or human accounts (access to UI) to perform specific tasks and given access policies.
    - In the Harbor UI, login with your credentials received at the end of the previous section.
    - Select `Administration` > `Users` and then the `NEW USER` button.


- In the Harbor UI, under `Administration` click `Registries` and then click the `New Endpoint` button. Fill out the following details in the form:
    - `Provider`: Select `Docker Hub` from the drop-down.
    - `Name`: Give the endpoint a name
    - `Description`: Optionally give it a description
    - `Endpoint URL`: Since Docker Hub was selected for Provider, the endpoint should default to `https://hub.docker.com`
    - `Access ID`: Use your Docker Hub credentials, you can use the same one as the one used during NKP deployment to bypass the rate limits.
    - `Access Secret`: Same as Access ID, use your secret.
    - `Verify Remote Cert`: Uncheck this.

- You can test the connection with the `Test Connection` button.
![Harbor Registry Config](./images/harbor-config.png)

- Hit `OK` to create.


- Now go to `Projects`
- Delete the default `library` project.
![Project Delete Library](./images/project-library-delete.png)

- We will now recreate that project. Click `NEW PROJECT` and fill in the following:
    - `Project Name`: Input `library` here
    - `Access Level`: Make sure this is unchecked for Public
    - `Project quota limits`: Leave at `-1` for unlimited here
    - `Proxy Cache`: Enable this and from the drop-down select the created Docker Hub registry we just created.
        - `Endpoint`: `https://hub.docker.com` should have been prepopulated by selecting the previous Docker Hub registry.
    - `Bandwidth`: Leave this at `-1`

![Project New](./images/project-library-new.png)

- Click `OK` to create the project.

- Once the `library` project has been created. Click that project and then select `Robot Accounts` and then the `NEW ROBOT ACCOUNT` button.
![Harbor Robot Account](./images/harbor-robot-account.png)

- Here fill out a name for the robot account. Set the duration of the robot account to `Never`. Hit `Next`.
- Under `Permissions` for ease of use (for now) click `Select All` and then `Finish`.
- In the subsequent screen will be the name of the Robot Account and secret. Make sure to write the credentials down/export to file.


## Installing Kyverno for use with Harbor as a Pull-through Cache
- **Context**: Normally if you wanted to use Harbor as an internal registry to cache your images you would need to either manually add the Harbor URL to all the Pod/Deployment definitions or manually update all the Kubernetes `daemon.json` with the Harbor registry and then restarting all their container runtime engine daemons. To circumvent having to do this we'll use a tool named `Kyverno` to enforce policies to use Harbor. In this example we'll use match and mutate policies.
- For more information about Kyverno visit [here](https://kyverno.io/docs/introduction/how-kyverno-works/)
- For a quick install for Kyverno run the following command:
```sh
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml
```
- Wait for all the resources to be created/deployed.
- Next create a new yaml file named `replace-image-registry-with-harbor.yaml` or the file in this repo [here](./artifacts/replace-image-registry-with-harbor.yaml)
- **NOTE**: Replace `<YOUR_HARBOR_URL>` in the file below with the Harbor URL retrieved from the earlier step (e.g. `10.54.114.152:5000`).
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: replace-image-registry-with-harbor
  annotations:
    policies.kyverno.io/title: Replace Image Registry With Harbor
    pod-policies.kyverno.io/autogen-controllers: none
    policies.kyverno.io/category: Sample
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    kyverno.io/kyverno-version: 1.11.4
    kyverno.io/kubernetes-version: "1.27"
    policies.kyverno.io/description: >-
      Some registries like Harbor offer pull-through caches for images from certain registries.
      Images can be re-written to be pulled from the redirected registry instead of the original and
      the registry will proxy pull the image, adding it to its internal cache.
      The imageData context variable in this policy provides a normalized view
      of the container image, allowing the policy to make decisions based on various 
      "live" image details. As a result, it requires access to the source registry and the existence
      of the target image to verify those details.      
spec:
  rules:
    - name: redirect-docker
      match:
        any:
          - resources:
              kinds:
                - Pod
              operations:
                - CREATE
                - UPDATE
      mutate:
        foreach:
          - list: request.object.spec.initContainers[]
            context:
              - name: imageData
                imageRegistry:
                  reference: "{{ element.image }}"
            preconditions:
              any:
                - key: "{{imageData.registry}}"
                  operator: Equals
                  value: index.docker.io
            patchStrategicMerge:
              spec:
                initContainers:
                  - name: "{{ element.name }}"
                    image: https://<YOUR_HARBOR_URL>/library/{{imageData.repository}}:{{imageData.identifier}}
          - list: request.object.spec.containers[]
            context:
              - name: imageData
                imageRegistry:
                  reference: "{{ element.image }}"
            preconditions:
              any:
                - key: "{{imageData.registry}}"
                  operator: Equals
                  value: index.docker.io
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    image: https://<YOUR_HARBOR_URL>/library/{{imageData.repository}}:{{imageData.identifier}}
```


- Now that we have Harbor setup we will need to setup the NKP cluster. First, we will need to create the Kubernetes credential for purposes of authenticating with Harbor. Run the following command replace the necessary values, replacing the `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` values with the robot account ones:
```sh
export REGISTRY_USERNAME=<your-robot-username>
export REGISTRY_PASSWORD=<your-robot-secret>
kubectl create secret generic harbor-registry-credentials \
 --from-literal username=$REGISTRY_USERNAME \
 --from-literal password=$REGISTRY_PASSWORD \
 --from-file=ca.crt=<(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d)
```


- We will now need to edit the Cluster YAML to update the image registry address to Harbor's URL
```sh
kubectl edit cluster <WORKLOAD_CLUSTER_NAME>
```
- This will open up the Cluster YAML in vi. Navigate to `spec.topology.variables.imageRegistries`:
    - Replace `url` with your Harbor URL appended with `/library`
    - Replace `imageRegistries.credentials.secretRef.name` value with the name of the credential we just created `harbor-registry-credentials`.
- Save and exit the yaml.
![Harbor Edit Cluster YAML](./images/harbor-cluster-yaml.png)

- Give this some time as editing the Cluster YAML will perform a rolling update as it brings up and down the various nodes in the cluster. You can watch the progress of this:
```sh
kubectl get nodes -w
```
- Example output:
```sh
NAME                                    STATUS                     ROLES           AGE     VERSION
jn-nkp-cluster-md-0-mdt98-2j8vp-ptjt2   Ready,SchedulingDisabled   <none>          4d      v1.31.4
jn-nkp-cluster-md-0-mdt98-2j8vp-rqr6n   Ready                      <none>          4d      v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-cvx7j   Ready                      <none>          9m11s   v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-qht8k   Ready                      <none>          50s     v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-tn9g2   Ready                      <none>          3m25s   v1.31.4
jn-nkp-cluster-pmxmb-8kq7f              Ready                      control-plane   4m29s   v1.31.4
jn-nkp-cluster-pmxmb-8r6cl              Ready                      control-plane   8m46s   v1.31.4
jn-nkp-cluster-pmxmb-gvcgp              NotReady                   control-plane   21s     v1.31.4
jn-nkp-cluster-pmxmb-hgllf              Ready                      control-plane   4d18h   v1.31.4
```

- The following is what it should look like when all nodes are ready:
```sh
kubectl get nodes
```

```sh
NAME                                    STATUS   ROLES           AGE     VERSION
jn-nkp-cluster-md-0-mdt98-pwlfz-cvx7j   Ready    <none>          18m     v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-qht8k   Ready    <none>          10m     v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-tn9g2   Ready    <none>          12m     v1.31.4
jn-nkp-cluster-md-0-mdt98-pwlfz-vpbwl   Ready    <none>          8m10s   v1.31.4
jn-nkp-cluster-pmxmb-8kq7f              Ready    control-plane   14m     v1.31.4
jn-nkp-cluster-pmxmb-8r6cl              Ready    control-plane   18m     v1.31.4
jn-nkp-cluster-pmxmb-gvcgp              Ready    control-plane   9m54s   v1.31.4
```



- Now to test whether the new registry works as a pull through. Run the following command to create a simple Deployment of nginx.
```sh
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```


## Adding Okta as Identity Provider to the Cluster

## Troubleshooting Errors