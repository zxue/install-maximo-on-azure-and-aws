# Install Maximo On Azure

The Maximo Application Suite allows users to sign on to a single, integrated platform to access key monitoring, maintenance, and
reliability applications across the business. Since the 8.x releases, MAS has been expanded to include Manage, the enterprise assets management application that exists in version 7.6.x and previous releases, and several new applications, Health, Monitor, Predict, Visual Inspection, IoT and the mobile app. While you can deploy MAS on-prem or in a public cloud, it requires Red Hat's OpenShift container platform as the underlying infrastructure. Below is a high-level Maximo architecture and core components.

![MAS Architecture](media/mas-architecture.png)

Before installing Maximo Application Suite (MAS 8.x or higher) on Microsoft Azure, check the [prerequisites](https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=azure-overview), incluidng Azure subscriptions, IBM Maximo license and entitlement key, Red Hat account and pull sceret, domain and subdomain names for your OpenShift cluster.

You can install Maximo using the BYOL option, or purchase Maximo from Azure Marketplace and then install it on Azure. With the BYOL option, you can install Maximo on a new OpenShift cluster, on an existing cluster, or 
1. New Red Hat® OpenShift® cluster by using the Installer Provisioned Infrastructure (IPI)
2. New Red Hat® OpenShift® cluster by using the User Provisioned Infrastructure (UPI)
3. Existing Red Hat® OpenShift® cluster

This document covers the third option, installing MAS on an existing OpenShift cluster through IBM TechZone.

