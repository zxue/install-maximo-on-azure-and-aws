# Install Maximo On Azure

The Maximo Application Suite allows users to sign on to a single, integrated platform to access key monitoring, maintenance, and
reliability applications across the business. Since the 8.x releases, MAS has been expanded to include Manage, the enterprise assets management application that exists in version 7.6.x and previous releases, and several new applications, Health, Monitor, Predict, Visual Inspection, IoT and the mobile app. While you can deploy MAS on-prem or in a public cloud, it requires Red Hat's OpenShift container platform as the underlying infrastructure. Below is a high-level Maximo architecture and core components.

![MAS Architecture](media/mas-architecture.png)

Before installing Maximo Application Suite (MAS 8.x or higher) on Microsoft Azure, check the [prerequisites](https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=azure-overview), including Azure subscriptions, IBM Maximo license and entitlement key, Red Hat account and pull secret, domain and subdomain names for your OpenShift cluster.

You can install Maximo using the BYOL option, or purchase Maximo from Azure Marketplace and then install it on Azure. With the BYOL option, you can install Maximo on a new OpenShift cluster, on an existing cluster, or 
1. New Red Hat® OpenShift® cluster by using the Installer Provisioned Infrastructure (IPI)
2. New Red Hat® OpenShift® cluster by using the User Provisioned Infrastructure (UPI)
3. Existing Red Hat® OpenShift® cluster

This document covers the third option, installing MAS on an existing OpenShift cluster. Note that the code has been tested on a docker container on MacBook using a 5 worker-node OpenShift cluster hosted through IBM TechZone. To deploy MAS on a new cluster, you will need to obtain a domain name, e.g. xyz.com, and a subdomain name, e.g. azureocp.xyz.com, for the OpenShift cluster.

## Install MAS Core

First, define the environment variables. 

```
export MAS_INSTANCE_ID=poc10
export IBM_ENTITLEMENT_KEY=xxx
export MAS_CONFIG_DIR=/mascli/masconfig
export SLS_LICENSE_ID=xxx
export SLS_LICENSE_FILE=/mascli/masconfig/license.dat
export UDS_CONTACT_EMAIL=xxx@xxx.com
export UDS_CONTACT_FIRSTNAME=firstname
export UDS_CONTACT_LASTNAME=lastname
export UDS_STORAGE_CLASS=ocs-storagecluster-ceph-rbd
export OCP_INGRESS_TLS_SECRET_NAME=xxx-ingress
export DB2_INSTANCE_NAME=db2inst
export MAS_APP_ID=manage 
```

- MAS_INSTANCE_ID is an arbitary string for the installation, e.g. inst1 
- IBM_ENTITLEMENT_KEY is the MAS license key from IBM.
- SLS_LICENSE_ID is a 12 digit hexadecimal number, e.g. 756A06D0C216. It is contained in the the MAS license key.
- MAS_CONFIG_DIR is the folder where the license keys are stored, e.g. masconfig, which is mapped to a docker container folder, e.g. /mascli/masconfig
- SLS_LICENSE_FILE is the license key file name and location, e.g. /mascli/masconfig/license.dat
- UDS_CONTACT_EMAIL can be your work or personal email address
- UDS_CONTACT_FIRSTNAME=firstname
- UDS_CONTACT_LASTNAME=lastname
- UDS_STORAGE_CLASS is the storage class used for MAS deployment, e.g. ocs-storagecluster-ceph-rbd
- OCP_INGRESS_TLS_SECRET_NAME is the ingress route tls certificate name. 
- DB2_INSTANCE_NAME is the db2 database instance name, e.g. db2inst
- MAS_APP_ID is manage if MAS Manage is to be installed

### Locate OpenShift Ingress Route Secret Name

The OCP_INGRESS_TLS_SECRET_NAME environment variable is optional in most cases, but it must be specified for the OpenShift cluster on Azure, mainly because it is a randomly assigned name. 

To find it, log in to the OpenShift cluster. Search "ingress" under Workloads | Secrets. Sort the list by Type. Note the name in the namespace of "openshift-ingress" and type of "kubernetes.io/tls", e.g. a85b5fa3-a6b3-4432-8b6d-b373773ce84e-ingress. Alternatively, you can locate the name using the `oc` command.

![OCP Ingress TLS Secret Name](media/ocp-ingress-tls-secret-name.png)

### Install ODF Storage Classes

Maximo Application Suite and its dependencies require storage classes that support ReadWriteOnce (RWO) and ReadWriteMany (RWX) access modes:
  - ReadWriteOnce volumes can be mounted as read-write by multiple pods on a single node.
  - ReadWriteMany volumes can be mounted as read-write by multiple pods across many nodes.

By default, an OpenShift cluster on Azure comes with two storage classes. While it is possible to use them to run the Ansible playbooks, it is much easier to work with OpenShift® Data Foundation (ODF), previously known as OpenShift Container Storage (OCS). 

  - Storage class (ReadWriteOnce): managed-csi
  - Storage class (ReadWriteMany): managed-premium

To install ODF, search "ODF" under Operators | OperatorHub from the OpenShift console. Choose the default options to complete the step. 

![OpenShift Data Foundation Install](media/ocp-odf-install.png)

Once ODF is installed, create a StorageSystem instance. Create 2 TiB or more storage on at least 3 worker nodes.

![OpenShift Storage System](media/odf-storage-system.png)

When the StorageSystem instance is created, three new storage classes are added. By default, the Ansible playbooks use the "
ocs-storagecluster-ceph-rbd" storage class.

![OpenShift Storage System](media/ocp-storage-classes.png)

### Start the Docker Container

Install docker on your computer, and run the docker command to launch the MAS cli instance. Alternatively, you can run Ansible playbooks locally, but the tradeoff is that you will need to install all dependencies, e.g python3. 

```
docker run -ti --rm --pull always -v ~/masconfig:/mascli/masconfig quay.io/ibmmas/cli
```

- ti: run the container interactively; allocate a pseud- TTY (pseudo terminals)​
- rm: remove the container when it exits​
- pull always: pull down image before running​
- v: bind mount a volume. Local machine folder “~/masconfig”; container folder: “/mascli/masconfig”​
- quay.io/ibmmas/cli: downloaded the mas cli image. Run "docker images" to see images available.

### Run Ansible Playbook to Install MAS Core

Log in to OpenShift and run the playbook to install MAS Core. This step may take one hour or longer.

```
oc login --token=xxx --server=https://api.xxx.westus.aroapp.io:6443

ansible-playbook ibm.mas_devops.oneclick_core

```

Alternatively, you can run the playbook locally on a remote host, as explained in the [Ansible](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html) documentation. The `-v` option is verbose mode (-vvv for more, -vvvv to enable connection debugging).

```
ansible-playbook ansible-devops/playbooks/oneclick_core.yml --connection=local -vvv
```

### Look Up MAS Admin Superuser Credentials

Navigate to the Secrets screen under Workloads from the OpenShift console. Search "superuser", and open the one in the namespace for the MAS installation, e.g. "mas-poc10-core".

![Look Up Superuser](media/lookup-superuser.png)

Copy the values for password and username. Note that the password appears first and username second. You will use them to log in to the Maximo administration app. Alternatively, use the `oc` command to look up the superuser credentials.

![Superuser Credentials](media/superuser-credentials.png)

### Download and Configure MAS Certificate

When navigating to the Maximo administration console in the browser, you are promoted with the "NET::ERR_CERT_AUTHORITY_INVALID". That is because the self-signed certificate is not trusted on your computer. 

![Maximo Admin URL Error](media/maximo-admin-url-error.png)

A quick workaround is that you press the "Advanced" button to continue and change "admin" to "api" in the URL address. You will see a screen with an error message that looks like the following. Change "api" back to "admin" in the URL address. You should land on the administration screen.

![Maximo API URL error](media/maximo-api-url-error.png)

For Maximo deployment on Azure, it is necessary that the certificate issue be addressed permanently. Go to Routes under Networking from the OpenShift console. Select the MAS namespace, e.g. mas-poc10-core, and open the admin route screen.

![Maximo Admin Route](media/maximo-admin-route.png)

Scroll down the screen to find the CA certificate. Copy the certificate and save it in a file, e.g. "ca.crt".

![Maximo Admin Route](media/maximo-admin-route-certificate.png)

On the MacBook, open the Keychain Access setting. Click the import items from the File menu. Locate the certificate file and import it. Select the imported item from the list, which is likely named something like "public.poc10.mas.imb.com". Double click on it and change the value of "when using this certificate" under Trust to "Always trust". Save the setting by entering your MacBook login password if promoted. You will notice that the icon next to the name from a red "x" to to a blue "+".

![Maximo Admin Route](media/maximo-admin-route-certificate-trust.png)

With that, you can open the Maximo administration application in the browser and log in without any certificate error prompt.

### Update User Data Service (UDS)

For Maximo deployment on Azure, you may notice an error from the command line that looks like the following. This error must be addressed before we activate the Manage application.

```
BAS configuration was unable to be verified: Connecting to BAS (verify=/tmp/bas.pem) at https://uds-endpoint-ibm-common-services.apps.bulqajcq.westus.aroapp.io failed: SSLError: HTTPSConnectionPool(host='uds-endpoint-ibm-common-services.apps.bulqajcq.westus.aroapp.io', port=443): Max retries exceeded with url: /v1/status (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get issuer certificate (_ssl.c:1131)')))
```

Log in to the Maximo administration console. Navigate to the admin with "/initialsetup", or click on the Configurations on the left side. We will need to update the URL, the AIP key, and the certificates. We will find these values from the OpenShift cluster.

![User Data Services](media/user-data-services.png)

Navigate to Secrets under Workloads from the OpenShift console. Search "event-api". Open the "event-api-secrets" screen, and copy the apikey value. Then open the "event-api-certs" screen, and copy the tls.crt value. Note that there are two certificates in the text. The first part includes all the text starting with the text "-----BEGIN CERTIFICATE-----" and ending with "-----END CERTIFICATE-----". The second part is the remaining text.

![OCP Event API Secrets](media/ocp-event-api-secrets.png)

Go back to the User Data Services (UDS) in the Maximo administration console. Open the edit screen.

- Replace the URL from the existing value, e.g. "https://uds-endpoint-ibm-common-services.apps.bulqajcq.westus.aroapp.io" to "".
- Replace the API key with the value you obtained previously.
- Delete three certificates named "part1", "part2" and "part3". Add the first certificate and name it "basCert1" using the first part of the certificates you obtained previously. Add the second certificate and name it "basCert2" using the second part of the certificates you obtained previously. Click "Confirm" to save the certificates. Click "Save" to save the changes.

The update of UDS settings may take 15 minutes and you will see a green status icon if successful. If it takes longer than that, chances are that the changes are not working and you will need to repeat the process with the correct values.

## Install DB2 and Activate MAS Manage

We are now ready to install and activate Maximo Manage. Run the Ansible playbook below. A DB2 database will be created automatically, and the Manage application will be deployed. This step can take two hours or longer.

```
ansible-playbook ibm.mas_devops.oneclick_add_manage
```

Note that if you want to use an existing database, you can skip this step and go the database configuration instead.

## Review and Connect to the Database

placeholder

## Install Demo Data for MAS Manage

placeholder


## Estimate Maximo License Requirements for Dev or Test Environment

Assuming that we install Maximo Manage only, with 2 administrators and 8 concurrent users, we will need approximately 150 AppPoints. 

| **Users**                    | **AppPoints** | **Quantity** | **Total** |
| ---------------------------- | ------------- | ------------ | --------- |
| Administrator user (Premium) | 15            | 2            | 30        |
| Application user (Premium)   | 15            | 8            | 120       |
|                              |               |              |           |
| Grand Total AppPoints        |               |              | **150**   |

Premium concurrent users consume 15 AppPoints, with the same login/logout logic.
- Premium users have all the privileges of both base and limited users, plus Manage Industry Solutions (O&G, Aviation, Transportation, Utilities, Nuclear, Civil Infrastructure) and add-ons, Predict, Health and Predict – Utilities, and Visual Inspection.
- Said in an easier way, premium users have access to everything in the MAS suite.
 
Premium administrator users consume 15 AppPoints.
- Application administrators administrate one or more applications, adds and assigns users to these applications, and uses the application-specific user interfaces to manage further user privileges.
- Suite administrator manages overarching system configuration settings from the suite administration pane.
 