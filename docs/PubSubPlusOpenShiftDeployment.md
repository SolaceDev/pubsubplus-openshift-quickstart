# Deploying a Solace PubSub+ Software Event Broker onto an OpenShift 4 Platform

Solace PubSub+ Software Event Broker meets the needs of big data, cloud migration, and Internet-of-Things initiatives, and enables microservices and event-driven architecture. Capabilities include topic-based publish/subscribe, request/reply, message queues/queueing, and data streaming for IoT devices and mobile/web apps. The event broker supports open APIs and standard protocols including AMQP, JMS, MQTT, REST, and WebSocket. As well, it can be deployed in on-premises datacenters, natively within private and public clouds, and across complex hybrid cloud environments.

This repository provides an example of how to deploy the Solace PubSub+ Software Event Broker onto an OpenShift 4 platform, including the steps to set up a Red Hat OpenShift Container Platform platform on AWS.



## Overview
There are [multiple ways](https://www.openshift.com/try ) to get to an OpenShift platform. This example uses the Red Hat OpenShift Container Platform for deploying an HA group of software event brokers, but the concepts are transferable to other compatible platforms. We also provide tips for how to set up a simple single-node deployment using [CodeReady Containers](https://developers.redhat.com/products/codeready-containers/overview ) (the equivalent of MiniShift for OpenShift 4) for development, testing, or proof of concept purposes.

The supported Solace PubSub+ Software Event Broker version is 9.7 or later.

For the Red Hat OpenShift Container Platform, we use a self-managed 60-day evaluation subscription of [RedHat OpenShift cluster in AWS](https://cloud.redhat.com/openshift/install#public-cloud ) in a highly redundant configuration, spanning three zones.

This repository expands on the [Solace Kubernetes Quickstart](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/README.md ) to provide an example of how to deploy Solace PubSub+ in an HA configuration on the OpenShift Container Platform running in AWS.

The event broker deployment does not require any special OpenShift Security Context; the default ["restricted" SCC](https://docs.openshift.com/container-platform/4.6/authentication/managing-security-context-constraints.html ) can be used.


### Related Information

You might also be interested in one of the following: 
- For a hands-on quick start using an existing OpenShift platform, refer to the [Quick Start guide](/README.md).
- For considerations about deploying in a general Kubernetes environment, refer to the [Solace PubSub+ on Kubernetes Documentation](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md)
- For the `pubsubplus` Helm chart configuration options, refer to the [PubSub+ Software Event Broker Helm Chart Reference](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/tree/master/pubsubplus#configuration).
- For OpenShift 3.11, refer to the [archived version of this project](https://github.com/SolaceProducts/pubsubplus-openshift-quickstart/tree/v1.1.1).


## Table of Contents
- [Production Deployment Architecture](#production-deployment-architecture)
- [Deployment Tools](#deployment-tools)
    - [Helm Charts](#helm-charts)
    - [OpenShift Templates](#openshift-templates)
- [Deploying Solace PubSub+ onto OpenShift / AWS](#deploying-solace-pubsub-onto-openshift--aws)
    - [Step 1: (Optional / AWS) Deploy a Self-Managed OpenShift Container Platform onto AWS](#step-1-optional--aws-deploy-a-self-managed-openshift-container-platform-onto-aws)
    - [Step 2: Specify an OpenShift Project for Deployment](#step-2-specify-an-openshift-project-for-deployment)
    - [Step 3: (Optional / ECR) Use a Private Image Registry](#step-3-optional--ECR-use-a-private-image-registry)
    - [Step 4, Option 1: Deploy Using Helm](#step-4-option-1-deploy-using-helm)
    - [Step 4, Option 2: Deploy Using OpenShift Templates](#step-4-option-2-deploy-using-openshift-templates)
- [Validating the Deployment](#validating-the-deployment)
    - [Viewing the Bringup logs](#viewing-the-bringup-logs)
- [Gaining Admin and SSH Access to the Event Broker](#gaining-admin-and-ssh-access-to-the-event-broker)
- [Testing Data Access to the Event Broker](#testing-data-access-to-the-event-broker)
- [Deleting a Deployment](#deleting-a-deployment)
    - [Delete the PubSub+ Deployment](#delete-the-pubsub-deployment)
    - [Delete the AWS OpenShift Container Platform Deployment](#deleting-the-aws-openshift-container-platform-deployment)
- [Using NFS for Persistent Storage](#using-nfs-for-persistent-storage)
- [Resources](#resources)


## Production Deployment Architecture

The following diagram shows an example of an HA group deployment of PubSub+ software event brokers in AWS:
![alt text](/docs/images/network_diagram.jpg "Network Diagram")

<br/>
The key parts to note in the diagram above are:
- the three PubSub+ Container instances in OpenShift pods, deployed on OpenShift (worker) nodes
- the cloud load balancer exposing the event router's services and management interface
- the OpenShift master nodes(s)
- the CLI console that hosts the `oc` OpenShift CLI utility client

## Deployment Tools

There are two options for tooling to use to deploy the Kubernetes cluster: Helm charts and OpenShift templates.

#### Helm Charts

The Kubernetes `Helm` tool allows great flexibility, allowing the process of event broker deployment to be automated through a wide range of configuration options including in-service rolling upgrade of the event broker. This example refers to the [Solace Kubernetes QuickStart project](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/tree/master ) for the Helm setting to use to deploy the event broker onto your OpenShift environment.

#### OpenShift Templates

You can directly use the OpenShift templates included in this project, without any additional tools, to deploy the event broker in a limited number of configurations. Follow the instructions for deploying using OpenShift templates in [Step 4, Option 2](#step-4-option-2-deploy-using-openshift-templates), below.


## Deploying Solace PubSub+ onto OpenShift / AWS

The following steps describe how to deploy an event broker onto an OpenShift environment. Optional steps are provided for:
- setting up a self-managed Red Hat OpenShift Container Platform on Amazon AWS infrastructure (marked as *Optional / AWS*) 
- using AWS Elastic Container Registry to host the Solace PubSub+ Docker image (marked as *Optional / ECR*).

**Tip:** You can skip Step 1 if you already have your own OpenShift environment available.

> Note: If you are using CodeReady Containers, follow the [getting started instructions](https://developers.redhat.com/products/codeready-containers/getting-started) to stand up a working CodeReady Containers deployment that supports Linux, MacOS, and Windows. At the `crc start` step it is helpful to: have a local `pullsecret` file created; specify CPU and memory requirements, allowing 1 CPU and 2.5 GiB memory for CRC internal purposes; specify a DNS server, for example: `crc start -p ./pullsecret -c 3 -m 8148 --nameserver 1.1.1.1`.

### Step 1: (Optional / AWS) Deploy a Self-Managed OpenShift Container Platform onto AWS

This step requires the following:
- a free Red Hat account. You can create one [here](https://developers.redhat.com/login ), if needed.
- a command console on your host platform with Internet access. The examples here are for Linux, but MacOS is also supported.
- a designated working directory for the OpenShift cluster installation. The automated install process creates files here that are required later for deleting the OpenShift cluster.
    ```
    mkdir ~/workspace; cd ~/workspace
    ```

To deploy the container platform in AWS, do the following:
1. If you haven't already, log in to your RedHat account.
2. On the [**Create an OpenShift cluster**](https://cloud.redhat.com/openshift/create) page, under **Run it yourself**, select **AWS**. 
3. On the [**Install OpenShift Container Platform 4**](https://cloud.redhat.com/openshift/install#public-cloud ) page, select **Installer-provisioned infrastructure**. A page is displayed that allows you to download the the required binaries and documentation.
3. Select your OS, and then make a note of the URL of the "Download installer" button.
4. On your host, in the command console, run the following commands to download and expand the OpenShift installer:
    ```
    wget <link-address>                        # Use the link address from the "Download installer" button
    tar -xvf openshift-install-linux.tar.gz    # Adjust the filename if needed
    rm openshift-install-linux.tar.gz
    ```
5. Run the utility to create an install configuration. Provide the necessary information at the prompts, including the Pull Secret from the RedHat instructions page. This will create the file `install-config.yaml` with the [installation configuration parameters](https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-customizations.html#installation-aws-config-yaml_installing-aws-customizations), most importantly the configuration for the worker and master nodes.
    ```
    ./openshift-install create install-config --dir=.
    ```
6. Edit the `install-config.yaml` file to update the worker node AWS machine type to meet the [minimum CPU and Memory requirements](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#cpu-and-memory-requirements) for the targeted PubSub+ Software Event Broker configuration. When you select an [EC2 instance type](https://aws.amazon.com/ec2/instance-types/), allow at least 1 CPU and 1 GiB memory for OpenShift purposes that cannot be used by the broker. The following is an example updated configuration:
    ```
    ...
    compute:
    - architecture: amd64
      hyperthreading: Enabled
      name: worker
      platform:
        aws:
          type: m5.2xlarge  # Adjust to your requirements
      replicas: 3
    ...
    ```
7. Create a backup copy of the configuration file, then launch the installation. The installation may take 40 minutes or more.
    ```
    cp install-config.yaml install-config.yaml.bak
    ./openshift-install create cluster --dir=.
    ```
8. If the installation is successful, information similar to the following is written to the command console. Take note of the web-console URL and login information for future reference.
    ```
    INFO Install complete!
    INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/opt/auth/kubeconfig'
    INFO Access the OpenShift web-console here: https://console-openshift-console.apps.iuacc.soltest.net
    INFO Login to the console with user: "kubeadmin", and password: "CKGc9-XUT6J-PDtWp-d4DSQ"
    ```
9. [Install](https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-default.html#cli-installing-cli_installing-aws-default) the `oc` client CLI tool.
10. Verify that your cluster is working correctly by following the hints from step 8, including verifying access to the OpenShift web-console.


### Step 2: Specify an OpenShift Project for Deployment

Create a new project or switch to your existing project (do not use the `default` project as its loose permissions don't reflect a typical OpenShift environment):
    ```
    oc new-project solace-pubsub    # adjust your project name as needed here and in subsequent commands
    ```

### Step 3: (Optional / ECR) Use a Private Image Registry

By default, the deployment scripts pull the Solace PubSub+ image from [Docker Hub](https://hub.docker.com/r/solace/solace-pubsub-standard/tags?page=1&ordering=last_updated). If the OpenShift worker nodes have Internet access, no further configuration is required.

However, if you need to use a private image registry, such as AWS ECR, you must supply a pull secret to enable access to the registry. The steps that follow show how to use AWS ECR for the broker image.

1. Download a free trial of the the Solace PubSub+ Enterprise Evaluation Edition by going to the **Docker** section of the [Solace Downloads](https://solace.com/downloads/?fwp_downloads_types=pubsub-enterprise-evaluation) page, or obtain an image from Solace Support.
2. Push the broker image to the private registry. Follow the specific procedures for the registry you are using. For ECR, see [Using Amazon ECR with the AWS CLI](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html).
    >Note: If you are advised to run `aws ecr get-login-password` as part of the "Authenticate to your registry" step and it fails, try running `$(aws ecr get-login --region <your-registry-region> --no-include-email)` instead.
    ![alt text](/docs/images/ECR-Registry.png "ECR Registry")
3. Create a pull secret from the registry information in the Docker configuration. This assumes that the ECR login happened on the same machine:
    ```
    oc create secret generic <my-pullsecret> \
       --from-file=.dockerconfigjson=$(readlink -f ~/.docker/config.json) \
       --type=kubernetes.io/dockerconfigjson
    ```
4. Use the pull secret you just created (`<my-pullsecret>`) in the deployment section, Step 4, below.

For additional information on private registries, see the [Using private registries](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#using-private-registries) section of the Solace Kubernetes Quickstart documentation.

#### Using CodeReady Containers
If you are using CodeReady Containers, you may need to perform a workaround if the ECR login fails on the console (e.g., on Windows). In this case, do the following:
1. Log into the OpenShift node: `oc get node` 
2. Run the `oc debug node/<reported-node-name>` command.
3. At the prompt, run the `chroot /host` command.
4. Run the following command:
    ````
    echo "<password-text>" | podman login --username AWS --password-stdin <registry>
    ````
    Where:
    - `<password-text>`: The text returned from the [`aws ecr get-login-password --region <ecr-region>`](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login-password.html) command. Run this command from a different machine where `aws` is installed (it is not straightforward to install the `aws` CLI on the CoreOS running on the node).
    - `<registry>`: The URI for your ECR registry, for example `9872397498329479394.dkr.ecr.us-east-2.amazonaws.com`.
5. Run `podman pull <your-ECR-image>` to load the image locally on the CRC node. 
    After you exit the node, you can use your ECR image URL and tag for the deployment. There is no need for a pull secret in this case.

### Step 4, Option 1: Deploy using Helm

Using Helm to deploy your cluster offers more flexibility in terms of event broker deployment options, compared to those offered by OpenShift templates (see [Option 2](#step-4-option-2-deploy-using-openshift-templates)).

Additional information is provided in the following documents:
- [Solace PubSub+ on Kubernetes Deployment Guide](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md)
- [Kubernetes Deployment Quick Start Guide](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/README.md)

This deployment uses PubSub+ Software Event Broker Helm charts. You can customize it by overriding the [default chart parameters](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/tree/master/pubsubplus#configuration).

Consult the [Deployment Considerations](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#pubsub-event-broker-deployment-considerations) section of the general Event Broker in Kubernetes Documentation when planning your deployment.

In particular, the `securityContext.enabled` parameter must be set to `false`, indicating not to use the provided pod security context but to let OpenShift set it using SecurityContextConstraints (SCC). By default OpenShift will use the "restricted" SCC.

By default the latest publicly available [Docker image](https://hub.Docker.com/r/solace/solace-pubsub-standard/tags/) of PubSub+ Standard Edition is used. Use a different image tag if required or [use an image from a different registry](#step-3-optional-use-a-private-image-registry). If you're using a different image, add the `image.repository=<your-image-location>,image.tag=<your-image-tag>` values (comma-separated) to the `--set` commands below. Also specify a pull secret if required: `image.pullSecretName=<my-pullsecret>`

The broker can be [vertically scaled](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#deployment-scaling ) using the `solace.size` chart parameter.

#### Steps:
1. Install Helm. Use the [instructions from Helm](//github.com/helm/helm#install), or if you're using Linux simply run the following command:
    ```bash
      curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```
     Helm is configured properly if the command `helm version` returns no error.
2. Follow one of the examples below to deploy your cluster.

    ##### For an _HA_ Deployment:
    ```bash
    # One-time action: Add the PubSub+ charts to local Helm
    helm repo add solacecharts https://solaceproducts.github.io/pubsubplus-kubernetes-quickstart/helm-charts
    # Initiate the HA deployment - specify an admin password
    helm install --name my-ha-release \
      --set securityContext.enabled=false,solace.redundancy=true,solace.usernameAdminPassword=<broker-admin-password> \
      solacecharts/pubsubplus
    # Check the notes printed on screen
    # Wait until all pods are running, ready, and the active event broker pod label is "active=true" 
    oc get pods --show-labels -w
    ```

    ##### For a Single-Node, _Non-HA_ Deployment (Using a _Pull_ _Secret_):
    ```bash
    # One-time action: Add the PubSub+ charts to local Helm
    helm repo add solacecharts https://solaceproducts.github.io/pubsubplus-kubernetes-quickstart/helm-charts
    # Initiate the non-HA deployment - specify an admin password
    helm install --name my-nonha-release \
      --set securityContext.enabled=false,solace.redundancy=true,solace.usernameAdminPassword=<broker-admin-password> \
      --set image.pullSecretName=<my-pullsecret> \
      solacecharts/pubsubplus
    # Check the notes printed on screen
    # Wait until the event broker pod is running, ready, and the pod label is "active=true" 
    oc get pods --show-labels -w
    ```

    **Note**: As an alternative to longer `--set` parameters, it is possible to define the same parameter values in a YAML file:
    ```yaml
    # Create example values file - specify an admin password
    echo "
    securityContext
      enabled: false
    solace
      redundancy: true,
      usernameAdminPassword: <broker-admin-password>" > deployment-values.yaml
    # Use values file
    helm install --name my-release \
      -v deployment-values.yaml \
      solacecharts/pubsubplus
    ```

### Step 4, Option 2: Deploy Using OpenShift Templates

This option use an OpenShift template and doesn't require Helm. This option assumes that you have completed [Step 2](#step-2-specify-an-openshift-project-for-deployment) and [Step 3](#step-3-optional-using-a-private-image-registry).

#### About the Template:
- You can copy templates files from the GitHub location to your local disk, edit them, and use them from there.
- Before you deploy, ensure that you determine your event broker [disk space requirements](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#disk-storage). The `BROKER_STORAGE_SIZE` parameter in the template has a default value of 30 gigabytes of disk space. You may need to update this value.
- By default, the template provisions a broker supporting 100 connections. You can adjust `export system_scaling_maxconnectioncount` in the template to increase the number of connections, but you must also ensure that adequate resources are available to the pod(s) by adjusting both `cpu` and `memory` requests and limits. For details, refer to the [System Resource Requirements](https://docs.solace.com/Configuring-and-Managing/SW-Broker-Specific-Config/System-Resource-Requirements.htm) in the Solace documentation.
- If using you are using [TLS to access broker services](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#enabling-use-of-tls-to-access-broker-services), you must configure a server key and certificate on the broker(s). Uncomment the related parts of the template file in your local copy and also specify a value for the `BROKER_TLS_CERT_SECRET` parameter.


#### Steps:

1. Define a strong password for the 'admin' user of the event broker, and then base64 encode the value:
    ```
    echo -n 'strong@dminPw!' | base64
    ```
    You will use this value as a parameter when you process the event broker OpenShift template.
2. Follow one of the examples below to deploy your cluster.


    ##### For a Single-Node, _Non-HA_ Deployment:
    This example uses all default values.  You can omit default parameters.

    ```
    oc process -f https://raw.githubusercontent.com/SolaceProducts/pubsubplus-openshift-quickstart/master/templates/eventbroker_singlenode_template.yaml \
        DEPLOYMENT_NAME=test-singlenode \
        BROKER_ADMIN_PASSWORD=<base64 encoded password> | oc create -f -
    # Wait until all pods are running and ready
    oc get pods -w --show-labels
    ```

    ##### For an _HA_ Deployment:
    In this example, we specify values for all parameters.

    The `BROKER_IMAGE_REGISTRY_URL` and `BROKER_IMAGE_TAG` parameters default to **solace/solace-pubsub-standard** and **latest**, respectively.

    ```
    oc process -f https://raw.githubusercontent.com/SolaceProducts/pubsubplus-openshift-quickstart/master/templates/eventbroker_ha_template.yaml \
        DEPLOYMENT_NAME=test-ha \
        BROKER_IMAGE_REGISTRY_URL=<replace with your Docker Registry URL> \
        BROKER_IMAGE_TAG=<replace with your Solace PubSub+ Docker image tag> \
        BROKER_IMAGE_REGISTRY_PULLSECRET=<my-pullsecret>
        BROKER_STORAGE_SIZE=30Gi \
        BROKER_TLS_CERT_SECRET=<my-tls-server-secret>  # See notes above \
        BROKER_ADMIN_PASSWORD=<base64 encoded password> | oc create -f -
    # Wait until all pods are running and ready
    oc get pods -w --show-labels
    ```

  
## Validating the Deployment

If you encounter any issues with your deployment, refer to the [Kubernetes Troubleshooting Guide](https://github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#troubleshooting) for help. Substitute any `kubectl` commands with `oc` commands. Before retrying a deployment, ensure to delete PVCs remaining from the unsuccessful deployment. Use the `oc get pvc` command to obtain a listing.

From the console, validate your deployment by running the following command:
```
$ oc get statefulset,service,pods,pvc,pv --show-labels
```
The output should look like the following:
```
NAME                                     DESIRED   CURRENT   AGE       LABELS
statefulset.apps/my-release-pubsubplus   3         3         2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/managed-by=Tiller,app.kubernetes.io/name=pubsubplus,helm.sh/chart=pubsubplus-1.0.0

NAME                                      TYPE           CLUSTER-IP     EXTERNAL-IP                                                                PORT(S)                                                                                                                                                             AGE       LABELS
service/my-release-pubsubplus             LoadBalancer   172.30.44.13   a7d53a67e0d3911eaab100663456a67b-73396344.eu-central-1.elb.amazonaws.com   22:32084/TCP,8080:31060/TCP,943:30321/TCP,55555:32434/TCP,55003:32160/TCP,55443:30635/TCP,80:30142/TCP,443:30411/TCP,5672:30595/TCP,1883:30511/TCP,9000:32277/TCP   2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/managed-by=Tiller,app.kubernetes.io/name=pubsubplus,helm.sh/chart=pubsubplus-1.0.0
service/my-release-pubsubplus-discovery   ClusterIP      None           <none>                                                                     8080/TCP,8741/TCP,8300/TCP,8301/TCP,8302/TCP                                                                                                                        2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/managed-by=Tiller,app.kubernetes.io/name=pubsubplus,helm.sh/chart=pubsubplus-1.0.0

NAME                          READY     STATUS    RESTARTS   AGE       LABELS
pod/my-release-pubsubplus-0   1/1       Running   0          2h        active=true,app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus,controller-revision-hash=my-release-pubsubplus-7b788f768b,statefulset.kubernetes.io/pod-name=my-release-pubsubplus-0
pod/my-release-pubsubplus-1   1/1       Running   0          2h        active=false,app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus,controller-revision-hash=my-release-pubsubplus-7b788f768b,statefulset.kubernetes.io/pod-name=my-release-pubsubplus-1
pod/my-release-pubsubplus-2   1/1       Running   0          2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus,controller-revision-hash=my-release-pubsubplus-7b788f768b,statefulset.kubernetes.io/pod-name=my-release-pubsubplus-2

NAME                                                 STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE       LABELS
persistentvolumeclaim/data-my-release-pubsubplus-0   Bound     pvc-7d596ac0-0d39-11ea-ab10-0663456a67be   30Gi       RWO            gp2            2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus
persistentvolumeclaim/data-my-release-pubsubplus-1   Bound     pvc-7d5c60e9-0d39-11ea-ab10-0663456a67be   30Gi       RWO            gp2            2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus
persistentvolumeclaim/data-my-release-pubsubplus-2   Bound     pvc-7d5f8838-0d39-11ea-ab10-0663456a67be   30Gi       RWO            gp2            2h        app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                 STORAGECLASS   REASON    AGE       LABELS
persistentvolume/pvc-58223d93-0b93-11ea-833a-0246f4c5a982   10Gi       RWO            Delete           Bound     openshift-infra/metrics-cassandra-1   gp2                      2d        failure-domain.beta.kubernetes.io/region=eu-central-1,failure-domain.beta.kubernetes.io/zone=eu-central-1c
persistentvolume/pvc-7d596ac0-0d39-11ea-ab10-0663456a67be   30Gi       RWO            Delete           Bound     solace-pubsub/data-my-release-pubsubplus-0    gp2                      2h        failure-domain.beta.kubernetes.io/region=eu-central-1,failure-domain.beta.kubernetes.io/zone=eu-central-1c
persistentvolume/pvc-7d5c60e9-0d39-11ea-ab10-0663456a67be   30Gi       RWO            Delete           Bound     solace-pubsub/data-my-release-pubsubplus-1    gp2                      2h        failure-domain.beta.kubernetes.io/region=eu-central-1,failure-domain.beta.kubernetes.io/zone=eu-central-1a
persistentvolume/pvc-7d5f8838-0d39-11ea-ab10-0663456a67be   30Gi       RWO            Delete           Bound     solace-pubsub/data-my-release-pubsubplus-2    gp2                      2h        failure-domain.beta.kubernetes.io/region=eu-central-1,failure-domain.beta.kubernetes.io/zone=eu-central-1b
[ec2-user@ip-10-0-23-198 ~]$
[ec2-user@ip-10-0-23-198 ~]$
[ec2-user@ip-10-0-23-198 ~]$ oc describe svc
Name:                     my-release-pubsubplus
Namespace:                solace-pubsub
Labels:                   app.kubernetes.io/instance=my-release
                          app.kubernetes.io/managed-by=Tiller
                          app.kubernetes.io/name=pubsubplus
                          helm.sh/chart=pubsubplus-1.0.0
Annotations:              <none>
Selector:                 active=true,app.kubernetes.io/instance=my-release,app.kubernetes.io/name=pubsubplus
Type:                     LoadBalancer
IP:                       172.30.44.13
LoadBalancer Ingress:     a7d53a67e0d3911eaab100663456a67b-73396344.eu-central-1.elb.amazonaws.com
Port:                     ssh  22/TCP
TargetPort:               2222/TCP
NodePort:                 ssh  32084/TCP
Endpoints:                10.131.0.17:2222
Port:                     semp  8080/TCP
TargetPort:               8080/TCP
NodePort:                 semp  31060/TCP
Endpoints:                10.131.0.17:8080
Port:                     semptls  943/TCP
TargetPort:               60943/TCP
NodePort:                 semptls  30321/TCP
Endpoints:                10.131.0.17:60943
Port:                     smf  55555/TCP
TargetPort:               55555/TCP
NodePort:                 smf  32434/TCP
Endpoints:                10.131.0.17:55555
Port:                     smfcomp  55003/TCP
TargetPort:               55003/TCP
NodePort:                 smfcomp  32160/TCP
Endpoints:                10.131.0.17:55003
Port:                     smftls  55443/TCP
TargetPort:               55443/TCP
NodePort:                 smftls  30635/TCP
Endpoints:                10.131.0.17:55443
Port:                     web  80/TCP
TargetPort:               60080/TCP
NodePort:                 web  30142/TCP
Endpoints:                10.131.0.17:60080
Port:                     webtls  443/TCP
TargetPort:               60443/TCP
NodePort:                 webtls  30411/TCP
Endpoints:                10.131.0.17:60443
Port:                     amqp  5672/TCP
TargetPort:               5672/TCP
NodePort:                 amqp  30595/TCP
Endpoints:                10.131.0.17:5672
Port:                     mqtt  1883/TCP
TargetPort:               1883/TCP
NodePort:                 mqtt  30511/TCP
Endpoints:                10.131.0.17:1883
Port:                     rest  9000/TCP
TargetPort:               9000/TCP
NodePort:                 rest  32277/TCP
Endpoints:                10.131.0.17:9000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Find the **'LoadBalancer Ingress'** value listed in the service description above. This is the publicly accessible Solace Connection URI for messaging clients and management. In the example, it is `a7d53a67e0d3911eaab100663456a67b-73396344.eu-central-1.elb.amazonaws.com`.

> **Note**: There is no Load Balancer support with CodeReady Containers. Services are accessed through NodePorts instead. To access the brokers, use the NodePort port numbers together with the CodeReady Containers' public IP addresses, which can be obtained by running the `crc ip` command.

### Viewing the Bringup Logs

To see the deployment events, navigate to:

- **OpenShift UI > (Your Project) > Applications > Stateful Sets > ((name)-pubsubplus) > Events**

You can access the log stack for individual event broker pods from the OpenShift UI, by navigating to:

- **OpenShift UI > (Your Project) > Applications > Stateful Sets > ((name)-pubsubplus) > Pods > ((name)-solace-(N)) > Logs**
    Where **(N)** above is the ordinal of the HA role of the PubSub+ broker:
      - 0 - Primary event broker
      - 1 - Backup event broker
      - 2 - Monitor event broker

![alt text](/docs/images/Solace-Pod-Log-Stack.png "Event Broker Pod Log Stack")


## Gaining Admin and SSH Access to the Event Broker

To access the event brokers, use the Solace Connection URI associated with the load balancer generated by the OpenShift template. As described in the introduction, you access the brokers through the load balancer service, which always point to the active event broker. The default port is 22 for CLI and 8080 for SEMP/[Solace PubSub+ Broker Manager](https://docs.solace.com/Solace-PubSub-Manager/PubSub-Manager-Overview.htm).

If you deployed OpenShift in AWS, then the Solace OpenShift QuickStart will have created an EC2 Load Balancer to front the event broker / OpenShift service.  The Load Balancer public DNS name can be found in the AWS EC2 console in the 'Load Balancers' section.

To launch Solace CLI or SSH into the individual event broker instances from the OpenShift CLI, use the following commands:

```
# CLI access
oc exec -it XXX-XXX-pubsubplus-X -- cli   # adjust pod name to your deployment
# shell access
oc exec -it XXX-XXX-pubsubplus-X -- bash  # adjust pod name to your deployment
```

You can also gain access to the Solace CLI and container shell for individual event broker instances from the OpenShift UI.  A web-based terminal emulator is available from the OpenShift UI. Navigate to an individual event broker Pod using the OpenShift UI:

- **OpenShift UI > (Your Project) > Applications > Stateful Sets > ((name)-pubsubplus) > Pods > ((name)-pubsubplus-(N)) > Terminal**

Once you have launched the terminal emulator to the event broker pod you can access the Solace CLI by executing the following command:

```
/usr/sw/loads/currentload/bin/cli -A
```

![alt text](/docs/images/Solace-Primary-Pod-Terminal-CLI.png "Event Broker CLI via OpenShift UI Terminal emulator")

See the [Solace Kubernetes Quickstart README](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#gaining-admin-access-to-the-event-broker ) for more details, including admin and SSH access to the individual event brokers.

## Testing Data Access to the Event Broker

A simple option for testing data traffic though the newly created event broker instance is the [SDKPerf tool](https://docs.solace.com/SDKPerf/SDKPerf.htm). Another option to quickly check messaging is [Try Me!](https://docs.solace.com/Solace-PubSub-Manager/PubSub-Manager-Overview.htm#Test-Messages), which is integrated into the [Solace PubSub+ Broker Manager](https://docs.solace.com/Solace-PubSub-Manager/PubSub-Manager-Overview.htm).

To try building a client, visit the Solace Developer Portal and select your preferred programming language to [send and receive messages](http://dev.solace.com/get-started/send-receive-messages/ ). Under each language there is a Publish/Subscribe tutorial that will help you get started.

>**Note**: The Host is the Solace Connection URI. It may be necessary to [open up external access to a port](//github.com/SolaceProducts/pubsubplus-kubernetes-quickstart/blob/master/docs/PubSubPlusK8SDeployment.md#modifying-or-upgrading-a-deployment ) used by the particular messaging API if it is not already exposed.

![alt text](/docs/images/solace_tutorial.png "getting started publish/subscribe")

<br>

## Deleting a Deployment
You can delete just the PubSub+ deployment, or tear down your entire AWS OpenShift Container Platform.

### Delete the PubSub+ Deployment

To delete the deployment or to start over from Step 4 in a clean state, do the following:

- If you used [Step 4, Option 1 (Helm)][#step-4-option-1-deploy-using-helm] to deploy, execute the following commands: 

    ```
    helm list            # lists the releases (deployments)
    helm delete XXX-XXX  # deletes instances related to your deployment - "my-release" in the example above
    ```

- If you used [Step 4, Option 2 (OpenShift templates)](#step-4-option-2-deploy-using-openshift-templates) to deploy, run the following:

    ```
    oc process -f <template-used> DEPLOYMENT_NAME=<deploymentname> | oc delete -f -
    ```

> **Note:** The commands above do not delete the dynamic Persistent Volumes (PVs) and related Persistent Volume Claims (PVCs). If you recreate the deployment with the same name and keep the original PVCs, the original volumes will be mounted with the existing configuration. 

To delete the PVCs (which also deletes the PVs), run the following commands:

```
# List PVCs
oc get pvc
# Delete unneeded PVCs
oc delete pvc <pvc-name>
```

To remove the project or to start over from [Step 2](#step-2-specify-an-openshift-project-for-deployment) in a clean state, delete the project using the OpenShift console or the command line: 
```
oc delete project solace-pubsub   # adjust your project name as needed
```
For more details, refer to the [OpenShift Projects](https://docs.openshift.com/enterprise/3.0/dev_guide/projects.html) documentation.

### Deleting the AWS OpenShift Container Platform Deployment

To delete your OpenShift Container Platform deployment that was set up at [Step 1](#step-1-optional--aws-deploy-a-self-managed-openshift-container-platform-onto-aws), run the following commands:

```
cd ~/workspace
./openshift-install help # Check options
./openshift-install destroy cluster
./openshift-install destroy bootstrap
```

You can now initiate the OpenShift stack deletion operation from the AWS CloudFormation console.


## Using NFS for Persistent Storage

The Solace PubSub+ supports NFS for persistent storage, with the "root_squash" option configured on the NFS server.

For an example deployment, specify the storage class from your NFS deployment ("nfs" in this example) in the `storage.useStorageClass` parameter and ensure `storage.slow` is set to `true`.

The Helm (NFS Server Provisioner)[https://github.com/helm/charts/tree/master/stable/nfs-server-provisioner] project is an example of a dynamic NFS server provisioner. Here are the steps to get going with it:


1. Create the required SCC:
    ```
    sudo oc apply -f https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs/deploy/kubernetes/scc.yaml
    ```
2. Install the NFS helm chart, which will create all dependencies:
    ```
    helm install stable/nfs-server-provisioner --name nfs-test --set persistence.enabled=true,persistence.size=100Gi
    ```
3. Ensure the "nfs-provisioner" service account got created:
    ```
    oc get serviceaccounts
    ```
4. Bind the SCC to the "nfs-provisioner" service account:
    ```
    sudo oc adm policy add-scc-to-user nfs-provisioner -z nfs-test-nfs-server-provisioner
    ```
5. Ensure the NFS server pod is up and running:
    ```
    oc get pod nfs-test-nfs-server-provisioner-0
    ```

If you're using a template to deploy, locate the volume mount for `softAdb` in the template and disable it by commenting it out:
    ```yaml
    # only mount softAdb when not using NFS, comment it out otherwise
    #- name: data
    #  mountPath: /usr/sw/internalSpool/softAdb
    #  subPath: softAdb
    ```

## Resources

For more information about Solace technology in general please visit these resources:

- The Solace Developer Portal website at: http://dev.solace.com
- Understanding [Solace technology.](http://dev.solace.com/tech/)
- Ask the [Solace community](http://dev.solace.com/community/).