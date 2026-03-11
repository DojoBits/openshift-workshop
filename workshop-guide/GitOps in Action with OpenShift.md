<img src="https://github.com/DojoBits/.github/blob/main/profile/assets/logo-name.png?raw=true" width="30%">


# Red Hat OpenShift GitOps Hands-On Workshop

Thank you for joining this DojoBits hands-on workshop about GitOps with OpenShfit.

In the workshop we'll learn:

- The foundation principles of GitOps
- Understand how the GitOps operational pattern evolves the principles of DevOps
- How to install and configure OpenShift development environment with OpenShift Local
- How to insrtall and use the OpenShift GitOps operator
- How to troubleshoot issues related to with workload devitions managed by GitOps.

Enjoy the workshop and please don't hesitate to reach out if you have any questions!

# 0. Prerequisites

The list of equipment prerequisites to complete the workshop are:

- web browser
- ssh client - openSSH, SecureCRT, MobaXterm,  putty, etc.
- internet connectivity to the AWS infrastructure
- local administrator rights

During the lab we'll be using port forwarding over ssh to access some of the services
running on the remote lab environment. To be able to access those services through
their urls in a web browser, we'll need to configure some static records in the
`hosts` file.

On Linux and MacOS:

```shell
sudo vim /etc/hosts && cat $_
```

On Windows:

- Open Notepad as Administrator:
  - Click the Start button (Windows icon).
  - Type Notepad in the search bar.
  - In the search results, right-click on "Notepad" and select "Run as administrator."
  - If a User Account Control (UAC) prompt appears, click "Yes" to grant permission.

- Open the Hosts File in Notepad:
  - In Notepad, click: `File > Open`.
  - Navigate to the following directory: `C:\Windows\System32\drivers\etc\`
  - In the "Open" dialog box, change the "Files of type" dropdown from
    `"Text Documents (*.txt)" to "All Files (*.*)`. This will make the "hosts"
    file visible.
  - Select the hosts file and click "Open"

Add the following records at the end of the hosts file and save it:

```shell
...
127.0.0.1 console-openshift-console.apps-crc.testing
127.0.0.1 openshift-gitops-server-openshift-gitops.apps-crc.testing
127.0.0.1 animation-app-animation-app.apps-crc.testing
127.0.0.1 api.crc.testing
127.0.0.1 oauth-openshift.apps-crc.testing
```

>*Note:* Please don't forget to revert those changes by the end of the hands-on

## 0.1 Get familiar with our Hands-On environment

The remote lab (hands-on) environment is a virtual machine deployed in AWS. We
use the latest Fedora Linux (Cloud Edition). Upon connection to the hands-on machine
with the defaut user `fedora` you'll have regular user privileges.
The `fedora` user is a member of the `sudoers` group. In case you need to elevate
you privilages, you could use the `sudo` command.


## 0.2 Connect to the Hands-On Environment

All hands-on exercises will be conducted in a dedicated virtual machine that can
be accessed through SSH.

Due to security reasons, SSH requires the private key file permissions to be set
to `400` (read-only for the owner). Let's do that first:

```shell
user@workstation:~$ chmod 400 key.pem
```

Next, let's login into our dedicated hands-on machine using an SSH client:

```shell
user@workstation:~$ ssh -i key.pem fedora@<Hands-On Environment IP>
~$
```

For the exercises that require remote service accesss, please open a separate SSH
session:

On Linux, MacOS and Windows (build-in OpenSSH client):

```shell
sudo ssh -L 6443:api.crc.testing:6443 \
         -L 443:console-openshift-console.apps-crc.testing:443 \
         -L 8443:openshift-gitops-server-openshift-gitops.apps-crc.testing:443 \
         -L 8080:animation-app-animation-app.apps-crc.testing:80 \
         -i key.pem fedora@<Hands-On Environment IP>
```

On Windows using the MobaXterm SSH Tunnels Manager (MobaSSHTunnel)

- Open MobaXterm.
- Access the SSH Tunnels Manager:
  - Go to the "Tools" menu at the top.
  - Select "MobaSSHTunnel (Port forwarding)".

Create a New SSH Tunnel:

- In the "MobaSSHTunnel" window, click the "New SSH tunnel" button.
- Configure the Tunnel:
  - Select the `Local port forwarding` radio button.
  - Forwarded port: This is the local_port on your Windows PC that MobaXterm
    will listen on (e.g., 8080).
  - SSH server: The IP address of the hands-on vm
  - SSH login: Your username on the ssh_server.
  - SSH port: The SSH port on the server (default is 22).
  - Remote server: The destination_address (from the SSH server's perspective)
    where the service you want to reach is running. Ex: `animation-app-animation-app.apps-crc.testing`
  - Remote port: The destination_port of the service on the remote server (e.g., 80, 6443, etc.)
  - Click: `Save`

In the MobaSSHTunnel dialog, click on the `key` icon for the desired tunnel to
select a specific SSH key that will be used for the authentication.


## 0.3 Install Red Hat OpenShift Local

For this hands-on we are going to use the Red Hat OpenShfit Local. This flavour
used to be known as Red Hat OpenShfit Code Ready Containers (CRC), hence the cli
client name.

Before starting with the installation, we must ensure that our Fedora hands-on
system meets the following requirements:

- Fedora (latest two releases recommended)
- At least 35GB free disk space in our home directory
- NetworkManager, libvirt, and QEMU (qemu-kvm) installed and running

```shell
sudo dnf install NetworkManager libvirt qemu-kvm -y
```

```shell
sudo systemctl enable --now libvirtd
```

According to the Red Hat OpenShift Local [install process](https://www.redhat.com/en/blog/install-openshift-local)

- We have to register for a free Red Hat account (if we don’t have one already)
- Go to the Red Hat [Console](https://console.redhat.com/openshift/create/local) and download:
  - The OpenShift Local (`crc`) tool for Linux
  - The Pull Secret file

>*Note:* If you don't have a Red Hat account already, don't worry! Ask your
>instructor for assistance.

<br>
<img src="./img/openshift-local-redhat-console.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Let's start by downloading the CRC cli tool:

```shell
mkdir tools && cd $_
curl -O -L https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
```

this will create a file `crc-linux-amd64.tar.xz` containing the crc cli. Next,
we'll extract the content of the archive setup the crc cli:

```shell
tar xvf crc-linux-amd64.tar.xz
mkdir -p ~/local/bin
mv crc-linux-*-amd64/crc ~/local/bin/
export PATH=$HOME/local/bin:$PATH
echo 'export PATH=$HOME/local/bin:$PATH' >> ~/.bashrc
```

Finally, let's check whether the crc cli works:

```shell
crc version
```

```shell
CRC version: 2.51.0+80aa80
OpenShift version: 4.18.2
MicroShift version: 4.18.2
```

So far so good!

Before we continue with the creation of the OpenShift Local vm, let's define some
configuration defaults:


```shell
crc config set consent-telemetry no
crc config set memory 18000
crc config set disk-size 50
```

```shell
crc config view
```

```shell
- consent-telemetry                     : no
- disk-size                             : 50
- memory                                : 18000
```

We are now ready to setup our OpenShift Local Virtual Machine. Let's use the
crc cli to configure the OpenShift environment and cache the VM images:

>*Note:* This step might take a few minutes as it require downloading the
> crcbundle - a compressed archive file that contains all the necessary components
> to set up and run the OpenShift cluster VM like container images and OpenShift
> disk image. In total it is about 5.74 GiB.



```shell
crc setup
```

```shell
INFO Using bundle path /home/fedora/.crc/cache/crc_libvirt_4.18.2_amd64.crcbundle
INFO Checking if running as non-root
INFO Checking if running inside WSL2
INFO Checking if crc-admin-helper executable is cached
INFO Caching crc-admin-helper executable
INFO Using root access: Changing ownership of /home/fedora/.crc/bin/crc-admin-helper-linux-amd64
INFO Using root access: Setting suid for /home/fedora/.crc/bin/crc-admin-helper-linux-amd64
INFO Checking if running on a supported CPU architecture
INFO Checking if crc executable symlink exists
INFO Creating symlink for crc executable
INFO Checking minimum RAM requirements
...
NFO Checking if CRC bundle is extracted in '$HOME/.crc'
INFO Checking if /home/fedora/.crc/cache/crc_libvirt_4.18.2_amd64.crcbundle exists
INFO Getting bundle for the CRC executable

INFO Downloading bundle: /home/fedora/.crc/cache/crc_libvirt_4.18.2_amd64.crcbundle...
5.74 GiB / 5.74 GiB [--------------------------------------------------------------------------------------------------------------------------------] 100.00% 6.61 MiB/s
INFO Uncompressing /home/fedora/.crc/cache/crc_libvirt_4.18.2_amd64.crcbundle
crc.qcow2:  20.25 GiB / 20.25 GiB [-----------------------------------------------------------------------------------------------------------------------------] 100.00%
oc:  176.52 MiB / 176.52 MiB [----------------------------------------------------------------------------------------------------------------------------------] 100.00%
Your system is correctly setup for using CRC. Use 'crc start' to start the instance
$
```

Finally, let's create an OpenShift instance:

>*Note*: OpenShift Local is designed to be run on local machines. Once you
> start your remote OpenShift Local instance over SSH please don't close the
> session as this could impact the operation of the OpenShift instance.

```shell
crc start
```

>*Note:* During the OpenShift instance creation you'll be asked for a `pull secret`
> If you don't have a Red Hat account please check with you instructor.

```shell
INFO Using bundle path /home/fedora/.crc/cache/crc_libvirt_4.18.2_amd64.crcbundle
INFO Checking if running as non-root
INFO Checking if running inside WSL2
INFO Checking if crc-admin-helper executable is cached
INFO Checking if running on a supported CPU architecture
INFO Checking if crc executable symlink exists
INFO Checking minimum RAM requirements
INFO Check if Podman binary exists in: /home/fedora/.crc/bin/oc
INFO Checking if Virtualization is enabled
INFO Checking if KVM is enabled
INFO Checking if libvirt is installed
INFO Checking if user is part of libvirt group
INFO Checking if active user/process is currently part of the libvirt group
INFO Checking if libvirt daemon is running
INFO Checking if a supported libvirt version is installed
INFO Checking if crc-driver-libvirt is installed
INFO Checking crc daemon systemd socket units
INFO Checking if vsock is correctly configured
INFO Loading bundle: crc_libvirt_4.18.2_amd64...
CRC requires a pull secret to download content from Red Hat.
You can copy it from the Pull Secret section of https://console.redhat.com/openshift/create/local.
? Please enter the pull secret
```

Paste the pull secret in the terminal and hit: Enter

```shell
? Please enter the pull secret ******************************************************************************************************************************************

WARN Cannot add pull secret to keyring: The name is not activatable
INFO Creating CRC VM for OpenShift 4.18.2...
INFO Generating new SSH key pair...
INFO Generating new password for the kubeadmin user
INFO Starting CRC VM for openshift 4.18.2...
INFO CRC instance is running with IP 127.0.0.1
...

INFO Operator console is progressing
INFO Operator console is progressing
INFO All operators are available. Ensuring stability...
INFO Operators are stable (2/3)...
INFO Operators are stable (3/3)...
INFO Adding crc-admin and crc-developer contexts to kubeconfig...
```

Once you see the previous message leave the SSH console opened and create a new
ssh session to your hands-on machine.


## 0.3.1 Verify the OpenShift Installation

In a fresh SSH session run the following command:

```shell
crc status
```

The status of the CRC VM should be `Running`:

```shell
CRC VM:          Running
OpenShift:       Running (v4.18.2)
Disk Usage:      25.49GB of 53.08GB (Inside the CRC VM)
Cache Usage:     28.13GB
Cache Directory: /home/fedora/.crc/cache
```


Now, let's try to open the OpenShift Local Web Console UI using a web browser. The web
console can be accessed on the following [url](https://console-openshift-console.apps-crc.testing/):

<br>
<img src="./img/openshift-kubeadmin-login.png" width="60%" style="display: block; margin: 0 auto;">
<br>



# 1. Red Hat OpenShift GitOps

## 1.1 Introduction to GitOps and its relation with DevOps

DevOps is a combination of cultural philosophies, practices, and tools that increases
an organization's ability to deliver applications and services at high velocity.
It's not just a set of tools or a specific methodology, but rather a cultural shift
that emphasizes collaboration and communication between software development (Dev)
and IT operations (Ops) teams.

The ultimate goal of DevOps is to shorten the systems development life cycle and
provide continuous delivery with high software quality, security, and reliability.

GitOps is a modern operational framework that extends the principles of DevOps
and Infrastructure as Code (IaC) to automate and manage the deployment, monitoring,
and lifecycle of applications and infrastructure.

<br>
<img src="./img/openshift-gitops.png" width="60%" style="display: block; margin: 0 auto;">
<br>

GitOps takes many of the core tenets of DevOps like automation, version control,
collaboration, and continuous delivery and applies them to infrastructure and
application deployments using Git repositories as the single source of truth for
declarative configurations (desired state). Automated tools continuously reconcile
the actual state of the system with the desired state defined in Git, ensuring
consistency and enabling automatic deployments, rollbacks, and an auditable history
of all changes.

- DevOps answers "Why?" and "What?"
  - Why should we collaborate more?
  - What practices  should we adopt (CI/CD, IaC, monitoring)?

- GitOps answers "How?"
  - How do we achieve continuous deployment and manage our infrastructure declaratively,
    with an auditable trail, and self-healing capabilities?


[GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) addresses several common challenges in software
development and operations:

- Manual and Error-Prone Deployments: Traditional manual deployments are susceptible
  to human error, leading to inconsistencies and failures. GitOps automates the
  entire deployment process, significantly reducing the risk of errors.

- Configuration Drift: Environments often diverge from their intended state due
  to ad-hoc changes or unmanaged updates. GitOps continuously reconciles the actual
  state of the cluster with the desired state defined in Git, automatically
  correcting any drift.

- Lack of Auditability and Traceability: Without a central, version-controlled
  record, it's difficult to track who made what changes, when, and why. Git provides
  a complete audit trail of every configuration change, enhancing transparency
  and accountability.

- Slow Deployment Cycles: Manual processes and complex pipelines can slow down
  the pace of deployments, hindering agile development. GitOps enables faster,
  more frequent, and reliable deployments by streamlining the entire release process.

- Complex Rollbacks: Reverting to a previous stable state can be challenging without
  proper versioning. With GitOps, a rollback is as simple as reverting a commit
  in Git, making recovery efficient and reliable.

- Security Concerns: Uncontrolled access to production environments can pose
  significant security risks. GitOps promotes a "pull-based" deployment model
  where agents in the cluster pull configurations from Git, eliminating the need
  for direct cluster access for human operators in most cases.


DevOps vs GitOps in a nutshell:

- Infrastructure as Code (IaC): Both DevOps and GitOps heavily rely on IaC. GitOps
  simply takes it a step further by mandating that all infrastructure and application
  configurations are stored declaratively in Git.

- Continuous Integration/Continuous Delivery (CI/CD): CI/CD pipelines are fundamental
  to DevOps. GitOps leverages these pipelines, particularly for the continuous
  delivery/deployment (CD) part, by triggering deployments based on changes in the
  Git repository.

- Collaboration and Auditability: DevOps emphasizes collaboration and transparency.
  GitOps enhances this by using Git's built-in features (pull requests, commit
  history, code reviews) for all changes, providing a clear audit trail and encouraging
  team collaboration on infrastructure and application deployments.

- Automation: Both methodologies prioritize automation to reduce manual errors and
  increase speed. GitOps automates the reconciliation process, ensuring the deployed
  state always matches the desired state in Git.

- Declarative vs. Imperative: DevOps can accommodate both imperative (step-by-step
  instructions) and declarative (desired state) approaches. GitOps specifically
  champions a declarative approach, which aligns well with modern cloud-native
  platforms like Kubernetes.

In summary GitOps isn't a replacement for DevOps, but rather a powerful, opinionated
implementation pattern that helps organizations mature their DevOps practices,
especially when dealing with cloud-native applications and highly automated
environments. It provides a structured and secure way to achieve continuous
deployment and infrastructure management within a DevOps framework.

## 1.2 OpenShift GitOps Operator overview

In OpenShift, GitOps is built around Argo CD as the declarative GitOps engine that
enables GitOps workflows across multicluster OpenShift and Kubernetes infrastructure.
Using Argo CD, we can sync the state of OpenShift and Kubernetes clusters and
applications with the content of the  Git repositories manually or automatically.

<br>
<img src="./img/openshift-gitops-argocd.png" width="60%" style="display: block; margin: 0 auto;">
<br>


> *Refresher:* An OpenShift Operator is a method of packaging, deploying, and managing
> a Kubernetes-native application. It extends the core Kubernetes (and by extension,
> OpenShift) functionality to automate the operational tasks typically performed
> by a human administrator for a specific application or service.

Argo CD continuously compares the state of the clusters and the Git repositories
to identify any drift and can automatically bring the cluster back to the desired
state if any change is detected on the Git repository or the cluster.

The auto-healing capabilities in Argo CD increase the security of the CD workflow
by preventing undesired, unauthorized, or unvetted changes that might be performed
directly on the cluster unintentionally or through security breaches.

Red Hat OpenShift GitOps is an add-on available as an operator in the OperatorHub
and can be installed with just a few clicks from the OpenShift Console. Once installed,
we can deploy Argo CD instances using kubernetes custom resource definitions.


> *Note:* In the OpenShift OperatorHub, we can find both a `community Argo CD` Operator
> and the `Red Hat OpenShift GitOps` Operator. While both leverage Argo CD at
> their core, there are significant differences in terms of support, integration,
> and additional features.

## 1.3 OpenShift GitOps Operator installation

Before we can use the OpenShift GitOps operator, we have have to install it - either
cluster-wide (available for all OpenShift projects) or in a specific project
(kubernetes namespace). Depending on the installation type - we'll need either
`cluster-admin` or `admin` privileges for the specific project (namespace).

> *Note:* During the initial installation of the GitOps operator, some Custom
> Resource Definitions (CRDs) needs to be created, which requrie cluster-admin
> privilegges.

Let's log-in into our OpenShift Local Web Console UI using the privileged `kubeadmin`
account. The web console can be accessed on the following [url](https://console-openshift-console.apps-crc.testing):

<br>
<img src="./img/openshift-kubeadmin-login.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Let's check what operators we have already installed in our OpenShift instance.
From the left sidebar navigate to `Operators` > `Installed Operators`:

<br>
<img src="./img/openshift-admin-installed-operators.png" width="70%" style="display: block; margin: 0 auto;">
<br>

Out of the box we have the Package Server operator installed, that enables OpenShift
to discover, display and make operators available in OperatorHub.

From the left sidebar navigate to `Operators` > `OperatorHub` and type in the
search field: `openshift gitops`:

<br>
<img src="./img/openshift-operatorhub-redhat-gitops.png" width="70%" style="display: block; margin: 0 auto;">
<br>

Select the `Red Hat OpenShift GitOps` operator by clicking on the card to view
the panel wiht additioanl details regarding the operator. From this pannel we can
select a specific version of the operator that we want to install, view the detailed
description and also some overview / summary infomration such as compatibility level,
privder, infrastructure features, etc:

<br>
<img src="./img/openshift-operatorhub-redhat-gitops-description.png" width="70%" style="display: block; margin: 0 auto;">
<br>

Review the operator details and click: `Install`. In the next screen `Install Operator`,
we can customize the GitOps operator deployment - Update Channel, version, installation
namespace, etc.

> *Note:* For production deployments it is recommended to:
-  Select the `Enable Operator recommended cluster monitoring on this Namespace`
   checkbox in order to enable the OpenShift's build-in monitoring stack based on
   Prometheus and Grafana for the operator namespace. This will also give us access
   to the pre-build Grafana Dashboards.
- Select `Manual` under Update Approval section, as that will it provide the
  required control for a phased rollout and robust testing.


For the deployment in our development environment, leave the default settings and
click `Install`:

<br>
<img src="./img/openshift-gitops-operator-installation.png" width="70%" style="display: block; margin: 0 auto;">
<br>

The installation process for the OpenShfit GitOps operator will start. It might
take up to a few minutes to complete depending on the environment:

<br>
<img src="./img/openshift-gitops-operator-installation-progress.png" width="70%" style="display: block; margin: 0 auto;">
<br>


In the OpenShift console navigate to `Operators` > `Installed Operators`. A new
entry for the "Red Hat OpenShfit GitOps" operator should appear with status
"Succeeded":

<br>
<img src="./img/openshift-gitops-operator-installed.png" width="70%" style="display: block; margin: 0 auto;">
<br>

Click on the the Red Hat OpenShift GitOps Operator name. In the Operator details
screen you can see additional details related to the operator like subscription
status, events, etc.


## 1.4 Logging into the ArgoCD Web UI

After we've successfully installed the OpenShift GitOps operator, it is time to
log-in into the ArgoCD console. To do that we'll need retrieve the credentials
automatically created during the installation.

From the quest (lab) vm console execute the following command:

```bash
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

The comment extracts the password for the ArgoCD admin account stored in the
openshift-gitops-cluster secret and base64 decode it prior printing it into the
console:

```text
nk8Dd******************wXhqP
$
```

Copy the password string and open a new browser tab on your machine (desktop or laptop)
and open the following url [ArgoCD Web UI](https://openshift-gitops-server-openshift-gitops.apps-crc.testing:8443)

> *Note:* We can access the ArgoCD Web UI using multiple ways. From the OpenShift
> Console Web UI, we can also click on the grid menu icon / App Switcher (the one
> representing a 3x3 matrix) and select the Cluster Argo CD link.

<img src="./img/openshift-app-switcher-argocd.png" width="70%" style="display: block; margin: 0 auto;">

> Because we are running OpenShift local in our lab on a remote machine and we
> do port forwarding over ssh that link won't work unless we specify the correct
> port number: 8443

In the ArgoCD log-in screen log-in using the credentials:

- username: admin
- password: <decoded password from the previous step>

<br>
<img src="./img/openshift-gitops-argocd-login.png" width="70%" style="display: block; margin: 0 auto;">
<br>

Upon successfull log-in we'll see the ArgoCD web console:

<br>
<img src="./img/openshift-argocd-web-console.png" width="70%" style="display: block; margin: 0 auto;">
<br>


## 1.5 Using the OpenShift GitOps Operator

In the previous tasks we've successfully installed the OpenShift GitOps operator
and we've been able to log-in into the ArgoCD Web UI. Now it is time to see how
ArgoCD can help us implement the GitOps principles in practice by deploying a
simple application which manifest is stored inside a Git repository.

>*Note:* Our example is based on the [bgd app](https://github.com/redhat-developer-demos/openshift-gitops-examples/tree/main/apps/bgd/base) (blue/green deployment) from
>the Red Hat Developer Demos GitHub repository.

## 1.5.1 Your Mission, should you choose to accept it

>*Note:* If you already have experience defining OpenShift Resources and creating
> ArgoCD application definitions, try solving the following challenge yourself,
> before moving on to the next section, where the solution is provided.

You are a member of a DevOps team that is tasked with managing mission critical
application in the organization production environment.

You've been asked by a member of the development team to help them deploy a
specific version of the company's flagship application on the OpenShift environment.
Ideally the development team should have the freedom to push updates to the
application and apply the changes to the infrastructure treaing the infrastructure
as code.

The application should be:

- deployed into a separate namespace: `animation-app`
- accessible over an OpenShift route called: `animation-app`
- be reachable through the route over `http` on port: `80`
- accessible inside the container on port (targetPort): `8080`
- defined as yaml mannifests in a Git repository
- treated as stateless application
- given an environment variable inside the container: `COLOR` with value `blue`
- already packaged into an OCI image using a CD pipeline and uploaded into an
  image repository: `quay.io/rhdevelopers/bgd:1.0.0`


## 1.5.2 Challenge Accepted

You've accepeted the challenge? That is great! Let's see how we can solve it.

We are asked to treat the infrastructure as code, so that we can easily colaborate,
version control and audit the changes made to the infrastructure and application.

>*Note:* We need to store the yaml manifests for the application definition inside
>a Git repository that ArgoCD can access. You could use your own Git repository
>or reference the source code provided in the workshop git repository.

>**Warning:** Please don't apply any of the yaml manifests for the application
> resources directly! We want to use ArgoCD to create them automatically for us.

We'll begin by creating a new `namespace` for the future application:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: animation-app
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
```

In the Namespace definition we've added the label: `argocd.argoproj.io/managed-by: openshift-gitops`
that has very important roles:

- Argo CD Management: The `argocd.argoproj.io/managed-by` key is a well-known
  label used by Argo CD itself. It's a signal to Argo CD that the resource it's
  attached to (in this case, our animation-app Namespace) is intended to be
  managed or observed by an Argo CD instance.

- Auto-Sync/Auto-Heal: It can signal to Argo CD that this namespace and related
  resources should be automatically discovered, reconciled, or healed if deviations
  from the Git source are detected by the openshift-gitops instance.

- Relates to specific Argo CD Instance: The value `openshift-gitops` specifies
  which particular Argo CD instance is responsible for managing this namespace.
  In OpenShift, `openshift-gitops` is the default namespace where the OpenShift
  GitOps Operator installs its Argo CD instance.

- Argo CD UI Filtering: In the Argo CD UI, you can often filter resources or
  applications based on labels. This label makes it clear which namespaces are
  under GitOps management and by which instance.

- Clarity for administrators: For human administrators, this label immediately
  identifies the namespace as being part of the GitOps deployment strategy managed
  by the OpenShift GitOps instance. It helps prevent manual changes that might
  conflict with the GitOps principles.


After the Namespace definition we have to decide how to manage the application pod.
A best practice for creating pods in Kubernetes (and OpenShift) is to use a controller
that will automatically manage the application pod - for example Deployment,
StatefulSet, DaemonSet, etc.

Our application is stateless, and a good choise is to use the Deployment controller:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: animation-app
  name: animation-app
  namespace: animation-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: animation-app
  strategy: {}
  template:
    metadata:
      labels:
        app: animation-app
    spec:
      containers:
      - image: quay.io/rhdevelopers/bgd:1.0.0
        name: animation-app
        env:
        - name: COLOR
          value: "blue"
```

The deployment will create the pods for our application, but in order to make
them accessible for the clients we need to provide a persitent network identity
by exposing them as a service:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: animation-app
  name: animation-app
  namespace: animation-app
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: animation-app
```

In the service definition, we've not explicitly specified any type. This means
that the service will be created as a `ClusterIP` service by default, which is
accessible only within our OpenShift cluster. To provide external access we'll
also need to create an OpenShift Route object:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: animation-app
  name: animation-app
  namespace: animation-app
spec:
  port:
    targetPort: 8080
  to:
    kind: Service
    name: animation-app
    weight: 100
```

A Route is a core component that allows us to expose a service running inside our
cluster to external (to OpenShift) clients. We can think of it as the entry point
for traffic coming from outside your OpenShift environment to our applications.

OpenShift clusters have a built-in "Router" (based on HAProxy by default, but we
can configure a custom one) that listens for external traffic and uses the defined
Routes to forward requests to the correct internal Service.

In this example we are not going to specify a custom Hostname for our route.
OpenShift will automatically generate one for us following the naming convention:

```text
<route-name>[-<namespace>].<default-domain>
```

We are almost done! So far we've discussed the standard resources requried to
deploy the application inside OpenShift. Next, let's create the ArgoCD application
object that will enable ArgoCD to automatically manage our application:

> *Note:* If you've already created an commited the above application resources
> into your own custom git repository, please update the respective fields about
> the source.path and source.repoURL!

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: animation-app
  namespace: openshift-gitops
spec:
  destination:
    namespace: animation-app
    server: https://kubernetes.default.svc
  project: default
  source:
    path: src/argo-app-deployment/animation-app
    repoURL: https://github.com/DojoBits/openshift-workshop
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```

This Application resource instructs Argo CD to:

- Monitor a specific directory (src/argo-app-deployment/animation-app) within a
  Git repository (https://github.com/DojoBits/openshift-workshop) on the main branch.
- Deploy the Kubernetes manifests found in that Git path to the animation-app
  namespace on the same Kubernetes cluster where Argo CD is running.
- Automatically synchronize the application's state, including pruning (deleting)
  resources that are no longer in Git, but not automatically self-healing
  (reverting manual changes) at all times.
- Ensure the target namespace (animation-app) is created if it doesn't already exist.


Let's dive a little bit into this resource definition:

- namespace: openshift-gitops: Defines the Kubernetes namespace where this
  Argo CD Application resource itself resides. In OpenShift, the openshift-gitops
  namespace is the default location where the Argo CD instance (and thus its
  application definitions) typically lives.

- apiVersion: argoproj.io/v1alpha1: Points to the API version for Argo CD's
  Application custom resource. It indicates that it's part of the argoproj.io
  group and is in its v1alpha1 (alpha) version.

- kind: Application: Explicitly states that this Kubernetes object is an Argo CD
  Application.

- metadata:
  - name: animation-app: This is the unique name for this Argo CD Application
    instance. This name is what you'll see in the Argo CD UI and use with the
    Argo CD CLI (e.g., argocd app get animation-app).
  - namespace: openshift-gitops: This is the Kubernetes namespace where this Argo
    CD Application resource itself resides. In OpenShift, the openshift-gitops
    namespace is the default location where the Argo CD instance (and thus its
    application definitions) typically lives.

- spec: This section defines the desired state and synchronization behavior of
        the application.

  - destination: Specifies where the application's resources should be deployed.
    - namespace: animation-app: The target Kubernetes namespace where the manifests
                 from the Git repository will be applied.
    - server: https://kubernetes.default.svc: This indicates the API server URL
              of the target Kubernetes cluster. https://kubernetes.default.svc is
              a common way to refer to the API server of the same cluster where
              Argo CD is running (often called the "in-cluster" destination).

  - project: default: Assigns the application to an Argo CD AppProject named default.
                      Argo CD Projects provide logical grouping for applications
                      and enforce security boundaries (e.g., controlling which
                      repositories and destination clusters/namespaces an application
                      can deploy to). The default project is typically pre-configured
                      to allow deployments to any namespace from any repository.

  - source: Defines where Argo CD should find the desired state of our
            application.
    - path: src/argo-app-deployment/animation-app: This is the specific directory
            within the Git repository where the Kubernetes YAML manifests (Deployment,
            Service, Route, etc.) for the animation-app are located.
    - repoURL: https://github.com/DojoBits/openshift-workshop: This is the URL of
            the Git repository that contains the application's configuration.
            Argo CD will clone and monitor this repository.
    - targetRevision: main: This specifies the Git branch (or tag, or commit SHA)
            that Argo CD should track. In our case, it's the main branch.

  - syncPolicy: This section controls how Argo CD performs synchronization.
    - automated: Enables automated synchronization.
      - prune: true: If true, Argo CD will automatically delete any Kubernetes
                    resources from the target cluster that are no longer defined
                    in the Git repository. This helps keep the cluster clean and
                    ensures Git is the single source of truth.
      - selfHeal: false: If false, Argo CD will not automatically revert any manual
                    changes made directly to the resources in the cluster. If you
                    manually change a resource (e.g., kubectl edit deployment),
                    Argo CD will detect that it's OutOfSync but won't automatically
                    revert it.
                    If true, it would automatically revert those manual changes
                    to match Git.
    - syncOptions: Provides additional options for the synchronization process.
      - CreateNamespace=true: This is a very useful option. It instructs Argo CD
                    to automatically create the animation-app namespace in the
                    target cluster if it doesn't already exist before attempting
                    to deploy resources into it.
                    This simplifies the bootstrap process.


Now, after we have all the resources needed to deploy the mission critical application,
is time to apply the ArgoCD Application definition. If you are not ussing your
own Git repository, you can find all the resources [here](https://github.com/DojoBits/openshift-workshop)

If you have'nt already ssh into the lab vm and create a new working directory:

```shell
$ mkdir animation-app && cd $_
$~/animation-app
```

Next, clone the git repository where the ArgoCD application definition is stored:

```shell
git clone git@github.com:DojoBits/openshift-workshop.git
```

```shell
Cloning into 'openshift-workshop'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 13 (delta 0), reused 13 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (13/13), done.
$~/animation-app
```

got into the new project directory:

```shell
cd openshift-workshop/
~/animation-app/openshift-workshop
```

The repository currently has the following structure:

```shell
.
├── README.md
└── src
    └── argo-app-deployment
        ├── animation-app
        │   ├── deployment.yaml
        │   ├── namespace.yaml
        │   ├── route.yaml
        │   └── service.yaml
        └── argo-app.yaml

4 directories, 6 files
```

Let's apply the ArgoCD application definition:

```shell
oc apply -f ./src/argo-app-deployment/argo-app.yaml
```

## 1.5.3 Validate the App deployment

We've just created a new app definition. Did it work? Let's find out.

Before we switch back to the OpenShift console and the ArgoCD web ui, let's
check the OpenShift resources part of our application from the shell:

```shell
oc -n animation-app get all
```

```shell
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
NAME                                 READY   STATUS    RESTARTS   AGE
pod/animation-app-6ff6b974f9-7rss6   1/1     Running   0          157m

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/animation-app   ClusterIP   10.217.5.84   <none>        8080/TCP   157m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/animation-app   1/1     1            1           157m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/animation-app-6ff6b974f9   1         1         1       157m

NAME                                     HOST/PORT                                      PATH   SERVICES        PORT   TERMINATION   WILDCARD
route.route.openshift.io/animation-app   animation-app-animation-app.apps-crc.testing          animation-app   8080                 None
```

So far it looks very promissing! ArgoCD has created for us the Namespace, Deployment,
service, and also the Route! Underneath the Host/Port column we can also see the
HOST name that was auto-generated for us by OpenShift.

We can also retreive the route by:

```shell
oc -n animation-app get route
```

```shell
NAME            HOST/PORT                                      PATH   SERVICES        PORT   TERMINATION   WILDCARD
animation-app   animation-app-animation-app.apps-crc.testing          animation-app   8080                 None
```

Lets test the route and see if it actually is going to redirct us to the application
and whether the application will respond:

```shell
curl http://animation-app-animation-app.apps-crc.testing
```

```shell
<!DOCTYPE html>
<html lang="en" >
<head>
  <meta charset="UTF-8">
  <title>CodePen - bubble animation</title>
  <link rel="stylesheet" href="./style.css">
</head>
<body>
<!-- partial:index.partial.html -->
<div id="app">
</div>
<!-- partial -->
<script  src="./script.js"></script>

</body>
</html>
```

Success! Our application is alive and well. Now we should be able to open an new
browser tab on our workstation (desktop/laptop) and try to reach the mission
critical application using the [route url](http://animation-app-animation-app.apps-crc.testing:8080/)

>*Note*: You might get a warning that the page is not secure. Configuring TLS
> termination and generation of signed certificates is currently outside of our
> scope

Wait a few seconds and you should see a number of flotting blue circles on your
screen:

<br>
<img src="./img/openshift-wellness-app.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Our mission critical wellness app is working! Watch and relax for a momoment :)


## 1.5.4 Review the Application resources using the OpenShift console

Job well done, but lets explore further our application inside OpenShift. Log-in
into the OpenShift Web Console and switch the perspective from `Administrator` to
`Developer` using the drop-down menu:

<br>
<img src="./img/openshift-developer-perspective.png" width="60%" style="display: block; margin: 0 auto;">
<br>

In the topology section select the `animation-app` from teh Project drop-down menu:

<br>
<img src="./img/openshift-topology-project.png" width="60%" style="display: block; margin: 0 auto;">
<br>

A visual representation of the Application Deployment as circle, with a number
represeting the current pod count (1) will appear in the center of the screen.
Click on the circle to see a side panel with additional details about the application
resources and status:


<br>
<img src="./img/openshift-application-topology.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Quests:

- explore the application resource definitions - replicaset, pod, service, etc.
- view the logs of the main container in the pod
- scale down the application down to 0 and then back to 1


## 1.5.5 Review the Application resources inside the ArgoCD

Let's explore the ArgoCD appliation we've created from the ArgoCD web UI. Switch
to you local browser tab with the ArgoCD UI and referesh.

> *Note:* In case you don't have it opened or you don't remember the credentials,
> please refer to task: `1.4 Logging into the ArgoCD Web UI` for detailed instructions.

You should see an Application Tile for the new ArgoCd application that we've
created:

<br>
<img src="./img/openshift-argocd-app-tile.png" width="60%" style="display: block; margin: 0 auto;">
<br>

In the tile we can see the status of our application:

- healthy: refers to the operational state and well-being of the deployed Kubernetes
  resources themselves in the cluster. It indicates whether the application is
  running as expected. Possible values: Healthy, Progressing, Degraded, Suspended,
  Missing, or Unknown
- synced: indicates wether the live state of all the Kubernetes resources (Deployments,
  Services, ConfigMaps, Secrets, etc.) managed by the Argo CD Application in our
  cluster exactly matches the desired state defined by the YAML manifests in our
  Git repository at the targetRevision specified in your Application resource.
  Possible values: Synced, OutOfSync, or Unknown


Click on the application tile to see the application details tree:

<br>
<img src="./img/openshift-argocd-application-tree.png" width="60%" style="display: block; margin: 0 auto;">
<br>

This view provides a graphical and hierarchical representation of all Kubernetes
resources managed by this specific Argo CD application, along with their sync and
health statuses.

Quests:
- explore the application resources - view their summary, parameters, events etc.
- what is the current state of our app? Is it healthy?


## 1.5.6 What could possibly go wrong?

So far we've seen how we can define our applications as code and how we can bridge
the gap between dev and ops by implementing the GitOps principles.

In the real world however, sometimes the resource `current state` can deverge from
the `desired state` due to various reasons:

- emergency fixes/hotfixes
- debugging and troubleshooting
- temporary scalling
- proof of concept or experimentation
- lack of process (GitOps) understanding
- many others...

No matter the reason, those deviations are quite often poorely communicated,
documented and sometimes remain as part of the infrastructure.

Let's see that in practice. Connect to the OpenShift development VM over ssh and
execute the following command to directly modify our application deployment adhoc:

```shell
oc -n animation-app set env deploy/animation-app COLOR=red
```

To validate the change go back to your local machine and refresh the browser tab
with the application [url](http://animation-app-animation-app.apps-crc.testing:8080/):

<br>
<img src="./img/openshift-app-modified-red.png" width="60%" style="display: block; margin: 0 auto;">
<br>

The behavior of our application has clearly beeing changed! Imagine a scenario
where bigger distributed team is troubleshooting a customer reported problem:

"the wellnesss app is no-longer calm and relaxing but causing anxiaty"

or even worse:

"After the change the application is completly broken"

Immediately multiple questions arise:

- was there a change?
- was the change the root cause for the outage?
- who did the change?
- why the change was made in a first place?
- what exactly was the changed?
- how could we restore the service?

OpenShift GitOps, powered by Argo CD, provides several powerful mechanisms to
protect against and manage deviations (drift) from your desired state in Git.
While it can't prevent every single direct manual change from happening (a
determined kubectl user can always try), it excels at detecting, reporting, and
automatically correcting these deviations, ensuring Git remains the single source
of truth.

Let's get back into the ArgoCD web ui and check the state of our application:

<br>
<img src="./img/openshift-argo-app-out-of-sync.png" width="60%" style="display: block; margin: 0 auto;">
<br>

This time we can clearly see the sync status shown as `OutOfSync`. Let's try
find out more details. Click on the application tile.

On the Application details tree we can see the states of all of the application
resources. Immediately it is visible tha there is a deviation in the sync state
of the application deployment:

<br>
<img src="./img/openshift-argo-app-tree-out-of-sync.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Click on the `animation-app` deploy resource highlighted with the orange mark. A
side pannel will open showing the resource details. Inside click on the "DIFF"
section and scrow down untill you notice the deviation:

>*Note:* You could also change the diff mode by selecting `Compact diff` or
>or `inline diff` mode.

<br>
<img src="./img/openshift-argo-app-diff.png" width="60%" style="display: block; margin: 0 auto;">
<br>

Now we know what exactly has change in our infrastructure out of band. Now, what
is the fastes way to restore it to our desired state?

With ArgoCD that is really straight-forward! Just click on the `sync` button in
upper right-hand side corner of the screen, and then in the pop-up sidebar click
on the `SYNCHRONIZE` button once more:

<br>
<img src="./img/openshift-argocd-app-synchronize.png" width="60%" style="display: block; margin: 0 auto;">
<br>

In few seconds our application is remediated and restored to its desired state.
Now the application sync status is green again. If we get back to the application
browser window we'll that the colore of the circles has been restored back to green.

The Outage is remediated!


# Thank you for attending the workshop!


_Copyright© 2025 DojoBits, all rights reserved_