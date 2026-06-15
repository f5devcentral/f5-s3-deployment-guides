# NetApp StorageGRID Cluster with F5 BIG-IP Local Traffic Manager

This guide explains how to deploy F5 BIG‑IP Local Traffic Manager (LTM) in front of a NetApp StorageGRID S3 cluster. It covers the network configuration, custom S3 health monitor, pool definition, and virtual server setup required to expose the cluster behind a single, highly available S3 endpoint.

This guide is intended for network engineers, system administrators, and storage operators who run a NetApp StorageGRID cluster and want to use F5 BIG-IP LTM to improve the performance, availability, and scalability of the S3 service. Readers should be familiar with basic L2/L3 networking concepts and have administrative access to both BIG-IP and the NetApp StorageGRID cluster.

> **Note:** This document is demonstration content intended to illustrate a reference deployment. Adapt it to your organization's standards and needs by substituting your own addresses, naming conventions, security policies, and operational practices before applying any of these steps in a production environment.

## Table of Contents

- [1. Setup](#1-setup)
  - [1.1 Overview](#11-overview)
  - [1.2 Networking](#12-networking)
- [2. BIG-IP Configuration](#2-big-ip-configuration)
  - [2.1 VLAN Configuration](#21-vlan-configuration)
  - [2.2 Configure Self IPs](#22-configure-self-ips)
  - [2.3 HTTP/HTTPS Custom S3 Health Monitor](#23-httphttps-custom-s3-health-monitor)
  - [2.4 Pool](#24-pool)
    - [2.4.1 Pool for S3 Traffic from BIG-IP to the NetApp StorageGRID Cluster](#241-pool-for-s3-traffic-from-big-ip-to-the-netapp-storagegrid-cluster)
  - [2.5 Optimization for S3 Traffic](#25-optimization-for-s3-traffic)
    - [2.5.1 TCP Profiles](#251-tcp-profiles)
      - [2.5.1.1 Client TCP Profile](#2511-client-tcp-profile)
      - [2.5.1.2 Server TCP Profile](#2512-server-tcp-profile)
  - [2.6 Virtual Server](#26-virtual-server)
    - [2.6.1 Virtual Server for S3 Client Traffic to BIG-IP](#261-virtual-server-for-s3-client-traffic-to-big-ip)
  - [2.7 Test the Virtual Server](#27-test-the-virtual-server)
    - [2.7.1 Configure the StorageGRID Cluster](#271-configure-the-storagegrid-cluster)
      - [2.7.1.1 Set Up a Tenant](#2711-set-up-a-tenant)
      - [2.7.1.2 Log In to the Tenant](#2712-log-in-to-the-tenant)
      - [2.7.1.3 Create a Bucket](#2713-create-a-bucket)
      - [2.7.1.4 Create a Test User](#2714-create-a-test-user)
      - [2.7.1.5 Create a Group for Test Bucket Access](#2715-create-a-group-for-test-bucket-access)
      - [2.7.1.6 Create an Access Key for the Test User](#2716-create-an-access-key-for-the-test-user)
    - [2.7.2 Use MinIO Warp to Test Connectivity to the StorageGRID S3 Cluster Through the BIG-IP Virtual Server](#272-use-minio-warp-to-test-connectivity-to-the-storagegrid-s3-cluster-through-the-big-ip-virtual-server)
  - [2.8 Automation with Ansible](#28-automation-with-ansible)

## 1. Setup

### 1.1 Overview

The lab environment used throughout this guide consists of the following components:

- F5 BIG-IP 21.1.0
- NetApp StorageGRID cluster
- MinIO Warp CLI tool for load testing

![Lab setup overview showing BIG-IP, NetApp StorageGRID cluster, and MinIO Warp client](./assets/setup_overview.png)

### 1.2 Networking

The table below lists the addresses used in the examples. Substitute your own addresses where appropriate.

| Component | VLAN     | Address       | Interface      |
| --------- | -------- | ------------- | -------------- |
| BIG-IP    | Internal | 10.150.91.130 | 1.1 (untagged) |
| BIG-IP    | External | 10.150.92.130 | 1.2 (untagged) |

![Lab network topology with BIG-IP internal and external VLANs connecting clients to the StorageGRID cluster](./assets/setup_networking.png)

## 2. BIG-IP Configuration

### 2.1 VLAN Configuration

Begin by adding VLANs to the BIG-IP. In the BIG-IP Configuration utility, go to **Network > VLANs** and click **Create**.
![BIG-IP VLAN list page with the Create button highlighted](./assets/bigip_vlan_create.png)

Create the **internal** VLAN:
![BIG-IP VLAN creation form for the internal VLAN](./assets/bigip_vlan_create_1_1.png)

- **Name:** `vlan1.1-internal`
- **Interface:** `1.1`
- **Tagging:** `Untagged`
- Click **Add**
- Click **Repeat**

This adds the **internal** VLAN. Now add the **external** VLAN in the same window:

- **Name:** `vlan1.2-external`
- **Interface:** `1.2`
- **Tagging:** `Untagged`
- Click **Add**
- Click **Finished**

The VLAN list should look like this:
![BIG-IP VLAN list showing the internal and external VLANs](./assets/bigip_vlan_overview.png)

### 2.2 Configure Self IPs

A Self IP is an IP address assigned to the BIG-IP on a specific VLAN. The BIG-IP uses Self IPs to source traffic and respond to ARP requests. Every VLAN that participates in client or server traffic needs at least one Self IP. Because this lab has two participating VLANs, internal (facing the StorageGRID cluster) and external (facing S3 clients), it requires two Self IPs, one per VLAN.

Continue by adding Self IPs that anchor the BIG-IP to each VLAN. In the BIG-IP Configuration utility, go to **Network > Self IPs** and click **Create**.

![BIG-IP Self IPs list page with the Create button highlighted](./assets/bigip_selfip.png)

First, create a Self IP for the internal network using the following values:

- **Name:** `Internal_Self_IP`
- **IP Address:** 10.150.91.130
- **Netmask:** 255.255.255.0
- **VLAN:** vlan1.1-internal

![New Self IP form populated with the internal interface values](./assets/bigip_selfip_create_internal_details.png)

Click **Repeat** to save the internal Self IP and open a fresh form for the external one, then fill in the following values:

- **Name:** `External_Self_IP`
- **IP Address:** 10.150.92.130
- **Netmask:** 255.255.255.0
- **VLAN:** vlan1.2-external

![New Self IP form populated with the external interface values](./assets/bigip_selfip_create_external_details.png)

Click **Finished** to save. Both Self IPs now appear in the list.

![Self IPs list showing both internal and external entries created](./assets/bigip_selfip_result.png)

### 2.3 HTTP/HTTPS Custom S3 Health Monitor

To verify that the StorageGRID S3 endpoint is reachable, create a custom monitor in BIG-IP. Go to **Local Traffic > Monitors** and click **Create**.
![BIG-IP custom monitor list with the Create button highlighted](./assets/bigip_monitor_create.png)

- **Type:** `HTTPS` or `HTTP`, depending on how your StorageGRID S3 endpoint is configured.
- **Name:** `Options_HTTPS_StorageGRID`
- **Send String:** `OPTIONS / HTTP/1.1\r\n\r\n`
- **Receive String:** `HTTP/1.1 200 OK`

![BIG-IP custom monitor form populated with the StorageGRID health check values](./assets/bigip_monitor_details.png)

Click **Finished** to save the new custom monitor.

### 2.4 Pool

A pool is a logical group of backend servers, called pool members, across which the BIG-IP load balances traffic. The BIG-IP runs a health monitor against each member and forwards traffic only to members that pass the monitor. In this guide, the pool members are the NetApp StorageGRID data nodes, and the BIG-IP distributes incoming S3 requests across them.

![Lab setup overview showing BIG-IP, StorageGRID cluster, and MinIO Warp client](./assets/setup_overview.png)

In the diagram above, the blue lines on either side of the BIG-IP represent the two S3 traffic segments. The lines on the left side show traffic between S3 clients and the BIG-IP. The lines on the right side show traffic between the BIG-IP and the NetApp StorageGRID nodes. Each segment can be unencrypted or encrypted, and you can decide this for each segment independently. Some organizations require storage traffic to be encrypted all the way from the S3 client to the S3 storage node, while others do not.

The **pool** configuration in this section controls the right-side segment: traffic between the BIG-IP and the NetApp StorageGRID data nodes. The left-side segment, between the S3 client and the BIG-IP, is controlled by the **virtual server** (see [2.6 Virtual Server](#26-virtual-server)). This guide provides steps for both unencrypted and encrypted options on each segment, so you can choose the combination that meets your organization's requirements.

#### 2.4.1 Pool for S3 Traffic from BIG-IP to the NetApp StorageGRID Cluster

Now, define the pool that represents the cluster. Navigate to **Local Traffic > Pools** and click **Create**.

![BIG-IP Pools list page with the Create button highlighted](./assets/bigip_pool.png)

Fill in the pool-level details:

- **Name:** `storageGRID_Pool`
- **Health Monitor:** `Options_HTTPS_StorageGRID`
- **Load Balancing Method:** Least Connections (member)

The `Options_HTTPS_StorageGRID` monitor created earlier sends an `OPTIONS` request to each node and marks the node healthy when it receives `HTTP/1.1 200 OK`.

In the **New Members** section, add the first StorageGRID node:

- **Name:** `storagenode-01`
- **Address:** 10.150.91.106
- **Service Port:** 18082 (the HTTPS S3 data service port for the StorageGRID cluster)

Click **Add** to save it to the member list. Then enter the next node's details and click **Add** again until all nodes are added. In this example, the StorageGRID cluster has three nodes, so add all three nodes in the cluster range 10.150.91.106 through 10.150.91.108 before completing the pool. Once every node appears in the member list, click **Finished**.

![New Pool form with the custom health monitor and StorageGRID node members added](./assets/bigip_pool_create_details.png)

The new pool now appears in the Pools list.

![Pools list showing the new storageGRID_Pool entry](./assets/bigip_pool_result.png)

### 2.5 Optimization for S3 Traffic

Tuning the BIG-IP configuration for the specific characteristics of S3 traffic can improve performance and efficiency. This section outlines some of the key optimizations to consider for an S3 workload, including connection limits and TCP profile settings. These optimizations are optional but can help you get the most out of your BIG-IP deployment in front of a NetApp StorageGRID cluster.

#### 2.5.1 TCP Profiles

A TCP profile controls how the BIG-IP manages TCP connections, such as timers, buffer sizes, and congestion control. The BIG-IP ships with built-in profiles, but the default values are not ideal for StorageGRID S3 traffic. For better performance, F5 engineers recommend creating two custom profiles: one for the client side (between the S3 client and the BIG-IP) and one for the server side (between the BIG-IP and the StorageGRID nodes). Each side has different needs, so the settings are tuned separately.

The following sections walk through both profiles. Start by navigating to **Local Traffic > Profiles > Protocol > TCP** and clicking **Create**.

![BIG-IP TCP profiles list page with the Create button highlighted](./assets/bigip_tcp_profile_create.png)

##### 2.5.1.1 Client TCP Profile

Set the name to `s3-tcp-custom-client` and set the parent profile to `s3-tcp`. This creates a new TCP profile that inherits all settings from the `s3-tcp` profile, which is optimized for S3 workloads, and allows you to customize selected values.

![New TCP profile form with the name s3-tcp-custom-client and the s3-tcp parent profile](./assets/bigip_tcp_profile_client_name.png)

In the **Timer Manager** section, set the **Minimum RTO** to `200` milliseconds. This reduces the minimum retransmission timeout, which can improve performance for S3 workloads that are sensitive to latency.

![Client TCP profile Timer Management section with Minimum RTO set to 200 milliseconds](./assets/bigip_tcp_profile_client_timer.png)

In the **Data Transfer** section, set **Nagle's Algorithm** to **Auto**. This allows the BIG-IP to decide when to enable or disable Nagle's algorithm based on traffic patterns, which can help optimize throughput for S3 workloads.

![Client TCP profile Data Transfer section with Nagle's Algorithm set to Auto](./assets/bigip_tcp_profile_client_data.png)

In the **Congestion Control** section, set **Congestion Control** to **CUBIC**. CUBIC is a modern congestion control algorithm that can improve performance in high-bandwidth, high-latency networks, which can benefit S3 traffic.

![Client TCP profile Congestion Control section set to CUBIC](./assets/bigip_tcp_profile_client_congestion.png)

Click **Finished** to save the client TCP profile.

##### 2.5.1.2 Server TCP Profile

Create another TCP profile for the server side with the name `s3-tcp-custom-server` and the parent profile set to `f5-tcp-lan`.

![New TCP profile form with the name s3-tcp-custom-server and the f5-tcp-lan parent profile](./assets/bigip_tcp_profile_server_name.png)

In the **Memory Management** section, set the following values:

- **Auto Receive Window** to **Enabled**
- **Auto Send Buffer** to **Enabled**
- **Proxy Buffer High** to `5200000` bytes
- **Proxy Buffer Low** to `5100000` bytes

![Server TCP profile Memory Management section with the auto window and proxy buffer values set](./assets/bigip_tcp_profile_server_memory.png)

In the **Loss Detection and Recovery** section, set the **Tail Loss** to **Disabled**.

![Server TCP profile Loss Detection and Recovery section with Tail Loss disabled](./assets/bigip_tcp_profile_server_loss.png)

Click **Finished** to save the server TCP profile.

### 2.6 Virtual Server

A virtual server is the listener that clients connect to. It binds an IP address and port on the BIG-IP to a pool, applies any configured profiles and policies, and forwards traffic to a healthy pool member according to the load balancing method. For this S3 deployment, the virtual server is the single endpoint that S3 clients target instead of talking to individual StorageGRID nodes.

As shown in the setup diagram in the [Pool section](#24-pool), the virtual server controls the top segment of traffic, between the S3 client(s) and the BIG-IP. Like the pool, the virtual server can be configured for unencrypted (HTTP) or encrypted (HTTPS) traffic, and this guide provides the steps for both. The two segments are independent: you can, for example, terminate TLS from clients at the BIG-IP while sending either encrypted or unencrypted traffic onward to the StorageGRID cluster.

#### 2.6.1 Virtual Server for S3 Client Traffic to BIG-IP

With the pool in place, create the virtual server that exposes it. Navigate to **Local Traffic > Virtual Servers** and click **Create**.

![BIG-IP Virtual Servers list page with the Create button highlighted](./assets/bigip_vs.png)

Fill in the basic details:

- **Name:** `StorageGRID_S3_Virtual_Server`
- **Source Address:** `0.0.0.0/0`
- **Destination Address/Mask:** 10.150.92.131 - the virtual IP (VIP) that S3 clients connect to
- **Service Port:** 443 (HTTPS)

![New Virtual Server form with name, destination address, and service port filled in](./assets/bigip_vs_create_details_1.png)

Next, configure the virtual server details:

- **1. Protocol:** Set to **TCP**
- **2. Protocol Profile (Client):** Set to the custom client TCP profile you created earlier (`s3-tcp-custom-client`)
- **3. Protocol Profile (Server):** Set to the custom server TCP profile you created earlier (`s3-tcp-custom-server`)
- **4. HTTP Client Profile:** Set to **HTTP**
- **5. SSL Profile (Client):** Set to the default client SSL profile (`clientssl`). Skip this if you want to use unencrypted HTTP between S3 clients and the BIG-IP.
- **6. SSL Profile (Server):** Set to the default server SSL profile (`serverssl`) if your StorageGRID cluster serves encrypted S3 traffic.
- **7. Source Address Translation:** Set to **Auto Map** to allow the BIG-IP to manage source address translation for return traffic.

![Virtual Server form with Source Address Translation set to Auto Map](./assets/bigip_vs_create_details_2.png)

In the **Resources** section, select the `storageGRID_Pool` pool from the **Default Pool** dropdown to bind the virtual server to the backend nodes.

Click **Finished** to save the virtual server.
![Virtual Server Resources section with storageGRID_Pool selected as the Default Pool](./assets/bigip_vs_create_details_3.png)

The virtual server now appears in the list. A green circle next to its name indicates that the virtual server is up and at least one pool member is passing health checks.

![Virtual Servers list showing StorageGRID_S3_Virtual_Server with a green available status](./assets/bigip_vs_result.png)

### 2.7 Test the Virtual Server

With the virtual server up and at least one healthy pool member, validate that S3 traffic flows end to end through the BIG-IP.

#### 2.7.1 Configure the StorageGRID Cluster

Before you test traffic, create an S3 bucket, a user, and an access key in the StorageGRID cluster.

![StorageGRID Dashboard](./assets/storagegrid-dashboard.png)

Click **Nodes** to review the cluster configuration:
![StorageGRID Nodes](./assets/storagegrid_nodes.png)

##### 2.7.1.1 Set Up a Tenant

In the left sidebar, select **Tenants** and click **Create**.
![StorageGRID Tenants page with the Create button highlighted](./assets/storagegrid_tenant_create.png)

In the wizard, enter **acme** as the tenant name and click **Continue**.
![StorageGRID Tenant Create - Name](./assets/storagegrid_tenant_create_details_1.png)

Select the **Allow platform services**, **Use own identity source**, and **Allow S3 Select** permission checkboxes, then click **Continue**.
![StorageGRID Tenant Create - Permissions](./assets/storagegrid_tenant_create_details_2.png)

Enter and confirm the password for the **root** user. Then click **Create tenant**.
![StorageGRID Tenant Create - Root Password](./assets/storagegrid_tenant_create_details_3.png)

A success message appears:
![StorageGRID Tenant Create - Success](./assets/storagegrid_tenant_create_result.png)

Click **Finish**.

##### 2.7.1.2 Log In to the Tenant

Select the tenant in which the test bucket will be created. In the sidebar, click **Tenants**, then select the newly created **acme** tenant:
![StorageGRID Tenant Select](./assets/storagegrid_tenant_select.png)

Click **Sign in**. You must sign in to the tenant to manage tenant resources:

![StorageGRID Tenant Login](./assets/storagegrid_tenant_signin.png)

In the sign-in window, enter **root** as the **Username** and enter the password you specified for the root user when creating the tenant. Click **Sign in**.
![StorageGRID Tenant Login](./assets/storagegrid_tenant_signin_password.png)

##### 2.7.1.3 Create a Bucket

From the tenant dashboard, click **Buckets**, then click **Create bucket**:
![StorageGRID tenant Buckets page with the Create bucket button highlighted](./assets/storagegrid_tenant_acme_create_bucket.png)

Enter **warp-connectivity-test** as the **Bucket name** and click **Continue**:
![StorageGRID bucket creation form with the bucket name entered](./assets/storagegrid_tenant_acme_create_bucket_details_1.png)

Scroll down through the features list until you see the **Create bucket** button, then click it:
![StorageGRID bucket creation features page](./assets/storagegrid_tenant_acme_create_bucket_details_2.png)

After the bucket is created, click **Finish**:
![StorageGRID bucket creation success message](./assets/storagegrid_tenant_acme_create_bucket_details_3.png)

##### 2.7.1.4 Create a Test User

To send traffic to the bucket, create a user with an access key. Click **Users**, then click **Create local user**:
![StorageGRID tenant Users page with the Create local user button highlighted](./assets/storagegrid_tenant_acme_create_user.png)

Enter **warp-test-user** in the **Full name** and **Username** fields. Enter and confirm a password in the **Password** and **Confirm password** fields. This user only needs data access to S3 buckets, so select **Yes** for the **Deny access** option. When the fields are complete, click **Continue**.
![StorageGRID local user properties form](./assets/storagegrid_tenant_acme_create_user_details_1.png)

Click **Create user** in the last wizard step:
![StorageGRID local user creation confirmation step](./assets/storagegrid_tenant_acme_create_user_details_2.png)

##### 2.7.1.5 Create a Group for Test Bucket Access

To restrict test users to access only the test bucket, create a security group. Click **Groups**, then click **Create group**:
![StorageGRID Tenant Create Security Group](./assets/storagegrid_tenant_acme_create_group.png)

Enter **warp-connectivity-test-group** as both the **Display name** and **Unique name**, then click **Continue**.
![StorageGRID Tenant Create Security Group](./assets/storagegrid_tenant_acme_create_group_details_1.png)

Select **Read-write** and click **Continue**:
![StorageGRID Tenant Create Security Group - Actions](./assets/storagegrid_tenant_acme_create_group_details_2.png)

Select **Custom** for **S3 group policy** and specify this policy:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::warp-connectivity-test",
        "arn:aws:s3:::warp-connectivity-test/*"
      ]
    }
  ]
}
```

Both resources are required. The bucket ARN covers bucket-level checks, and the object ARN covers the objects MinIO Warp writes, reads, and deletes during the mixed workload.

Click **Continue** when finished.
![StorageGRID Tenant Create Security Group - Policy](./assets/storagegrid_tenant_acme_create_group_details_3.png)

The last step is to assign the user to the group. In the dialog, select **warp-test-user** and click **Create group**.
![StorageGRID Tenant Create Security Group - Users](./assets/storagegrid_tenant_acme_create_group_details_4.png)

The group is created:
![StorageGRID Tenant Create Security Group - Users](./assets/storagegrid_tenant_acme_create_group_result.png)

##### 2.7.1.6 Create an Access Key for the Test User

To allow the user to access the S3 bucket programmatically, create an **Access key**. Select **Users**, then select **warp-test-user** from the users list:
![StorageGRID tenant Users page with warp-test-user selected](./assets/storagegrid_tenant_acme_access_key_user.png)

Select the **Access keys** tab and click **Create key**:
![StorageGRID user Access keys tab with the Create key button highlighted](./assets/storagegrid_tenant_acme_access_key_details_1.png)

Select **Set an expiration time**, then choose the next day at **12:00 AM**. _Do not create test keys without an expiration date._ Click **Create access key**:
![StorageGRID access key creation form with an expiration time selected](./assets/storagegrid_tenant_acme_access_key_details_2.png)

Copy and save the **Access key ID** and **Secret access key**. Click **Finish**.
![StorageGRID access key values page showing the access key ID and secret access key](./assets/storagegrid_tenant_acme_access_key_details_3.png)

The created access key will appear in the user's access key list:
![StorageGRID user access key list showing the created access key](./assets/storagegrid_tenant_acme_access_key_result.png)

#### 2.7.2 Use MinIO Warp to Test Connectivity to the StorageGRID S3 Cluster Through the BIG-IP Virtual Server

MinIO Warp is MinIO's S3 benchmarking tool. It drives a representative S3 workload (mixed PUT, GET, DELETE, and STAT operations) against an endpoint and reports throughput, latency, and error counts. Pointing MinIO Warp at the BIG-IP virtual server address, rather than at an individual StorageGRID node, confirms that traffic flows end to end through the load balancer and gives a baseline measurement of the cluster's performance behind the BIG-IP.

Download and install the MinIO Warp CLI from the official MinIO download page:

[https://www.min.io/download/minio-warp](https://www.min.io/download/minio-warp)

Configure MinIO Warp with the BIG-IP virtual server address as the S3 endpoint, along with the access key ID and secret access key generated on the StorageGRID cluster, and run a short mixed-workload test:

```bash
export WARP_HOST=10.150.92.131:443               # the BIG-IP virtual server address and port
export WARP_ACCESS_KEY=your-access-key-id        # the access key ID saved earlier
export WARP_SECRET_KEY=your-secret-access-key    # the secret access key saved earlier

./warp mixed \
  --host "$WARP_HOST" \
  --access-key "$WARP_ACCESS_KEY" \
  --secret-key "$WARP_SECRET_KEY" \
  --bucket warp-connectivity-test \
  --objects 10 \
  --obj.size 1KiB \
  --concurrent 1 \
  --duration 2s \
  --tls \
  --insecure \
  --json > test_result.json
```

When the run finishes, use `jq` to summarize the totals from the JSON report. Inspect the full output with `cat test_result.json` if you need a detailed breakdown.

```bash
jq -r '
  .total |
  "Requests: \(.total_requests) | Objects: \(.total_objects) | Bytes: \(.total_bytes) | Errors: \(.total_errors)"
' test_result.json


Requests: 12 | Objects: 12 | Bytes: 8192 | Errors: 0
```

An error count of zero confirms that every request reached the StorageGRID cluster and returned a successful response through the virtual server.

### 2.8 Automation with Ansible

The manual steps above can be reproduced declaratively with Ansible. The playbooks, roles, and inventory live in [automation/](./automation/). See [automation/README.md](./automation/README.md) for installation, credentials, and usage.
