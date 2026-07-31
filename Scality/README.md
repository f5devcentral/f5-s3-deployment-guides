# Scality RING Cluster with F5 BIG-IP Local Traffic Manager

This guide explains how to deploy F5 BIG‑IP Local Traffic Manager (LTM) in front of a Scality RING cluster. It covers the network configuration, custom S3 health monitor, pool definition, and virtual server setup required to expose the cluster behind a single, highly available S3 endpoint.

This guide is intended for network engineers, system administrators, and storage operators who run a Scality RING cluster and want to use F5 BIG-IP LTM to improve the performance, availability, and scalability of the S3 service. Readers should be familiar with basic L2/L3 networking concepts and have administrative access to both BIG-IP and the Scality RING cluster.

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
  - [2.5 Optimization for S3 Traffic](#25-optimization-for-s3-traffic)
  - [2.6 Virtual Server](#26-virtual-server)
  - [2.7 Test the Virtual Server](#27-test-the-virtual-server)
    - [2.7.1 Configure the RING Cluster](#271-configure-the-ring-cluster)
    - [2.7.2 Use Warp to Test Connectivity to the RING S3 Cluster Through the BIG-IP Virtual Server](#272-use-warp-to-test-connectivity-to-the-ring-s3-cluster-through-the-big-ip-virtual-server)
  - [2.8 Automation with Ansible](#28-automation-with-ansible)

## 1. Setup

### 1.1 Overview

The lab environment used throughout this guide consists of the following components:

- F5 BIG-IP 21.1.0
- Scality RING cluster
- Warp CLI tool for load testing

![Lab setup overview showing BIG-IP, Scality RING cluster, and S3 clients](./assets/setup_overview.png)

### 1.2 Networking

The table below lists the addresses used in the examples. Substitute your own addresses where appropriate.

| Component | VLAN             | Address          | Interface      |
| --------- | ---------------- | ---------------- | -------------- |
| BIG-IP    | vlan1.1-internal | 10.150.91.145/24 | 1.1 (untagged) |
| BIG-IP    | vlan1.2-external | 10.150.92.145/24 | 1.2 (untagged) |

![Lab network topology with BIG-IP internal and external VLANs connecting clients to the RING cluster](./assets/setup_networking.png)

## 2. BIG-IP Configuration

### 2.1 VLAN Configuration

Begin by adding VLANs to the BIG-IP. In the BIG-IP Configuration utility, go to **Network (1) > VLANs (2)** and click **Create (3)**.
![BIG-IP VLAN list page with the Create button highlighted](./assets/bigip_vlan_create.png)

Create the **internal** VLAN:
![BIG-IP VLAN creation form for the internal VLAN](./assets/bigip_vlan_create_1_1.png)

- **Name (1):** `internal`
- **Interface (2):** `1.1`
- **Tagging (2):** `Untagged`
- Click **Add (3)**
- Click **Repeat (4)**

This adds the **internal** VLAN. Now add the **external** VLAN in the same window:

- **Name:** `external`
- **Interface:** `1.2`
- **Tagging:** `Untagged`
- Click **Add**
- Click **Finished**

The VLAN list should look like this:
![BIG-IP VLAN list showing the internal and external VLANs](./assets/bigip_vlan_overview.png)

### 2.2 Configure Self IPs

A Self IP is an IP address assigned to the BIG-IP on a specific VLAN. The BIG-IP uses Self IPs to source traffic and respond to ARP requests. Every VLAN that participates in client or server traffic needs at least one Self IP. Because this lab has two participating VLANs, internal (facing the RING cluster) and external (facing S3 clients), it requires two Self IPs, one per VLAN.

Continue by adding Self IPs that anchor the BIG-IP to each VLAN. In the BIG-IP Configuration utility, go to **Network (1) > Self IPs (2)** and click **Create (3)**.

![BIG-IP Self IPs list page with the Create button highlighted](./assets/bigip_selfip.png)

First, create a Self IP for the internal network using the following values:

- **Name (1):** `internal_self_ip`
- **IP Address (2):** 10.150.91.145
- **Netmask (3):** 255.255.255.0
- **VLAN (4):** internal

![New Self IP form populated with the internal interface values](./assets/bigip_selfip_create_internal_details.png)

Click **Repeat (5)** to save the internal Self IP and open a fresh form for the external one, then fill in the following values:

- **Name (1):** `external_self_ip`
- **IP Address (2):** 10.150.92.145
- **Netmask (3):** 255.255.255.0
- **VLAN (4):** external

![New Self IP form populated with the external interface values](./assets/bigip_selfip_create_external_details.png)

Click **Finished (5)** to save. Both Self IPs now appear on the list.

![Self IPs list showing both internal and external entries created](./assets/bigip_selfip_result.png)

### 2.3 HTTP/HTTPS Custom S3 Health Monitor

To verify that the RING endpoint is reachable, create a custom monitor in BIG-IP. Go to **Local Traffic (1) > Monitors (2)** and click **Create (3)**.
![BIG-IP custom monitor list with the Create button highlighted](./assets/bigip_monitor_create.png)

- **Type (1):** `HTTPS` or `HTTP`, depending on how your RING endpoint is configured.
- **Name (2):** `Health_HTTP_RING`
- **Send String (3):** `GET /_/healthcheck/deep HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n`
- **Receive String (4):** `HTTP/1.1 200 OK`

![BIG-IP custom monitor form populated with the RING health check values](./assets/bigip_monitor_details.png)

Click **Finished (5)** to save the new custom monitor.

### 2.4 Pool

A pool is a logical group of backend servers, called pool members, across which the BIG-IP load balances traffic. The BIG-IP runs a health monitor against each member and forwards traffic only to members that pass the monitor. In this guide, the pool members are the Scality RING data nodes, and the BIG-IP distributes incoming S3 requests across them.

![Lab setup overview showing BIG-IP, RING cluster, and S3 clients](./assets/setup_overview.png)

In the diagram above, the blue lines on either side of the BIG-IP represent the two S3 traffic segments. The lines on the left side show traffic between S3 clients and the BIG-IP. The lines on the right side show traffic between the BIG-IP and the Scality RING nodes. Each segment can be unencrypted or encrypted, and you can decide this for each segment independently. Some organizations require storage traffic to be encrypted all the way from the S3 client to the S3 storage node, while others do not.

The pool configuration in this section controls the right-side segment: traffic between the BIG-IP and the Scality RING data nodes. The left-side segment, between the S3 client and the BIG-IP, is controlled by the virtual server (see [2.6 Virtual Server](#26-virtual-server)). This guide provides steps for both unencrypted and encrypted options on each segment, so you can choose the combination that meets your organization's requirements.

#### 2.4.1 Pool for S3 Traffic from BIG-IP to the Scality RING Cluster

Now, define the pool that represents the cluster. Navigate to **Local Traffic (1) > Pools (2)** and click **Create (3)**.

![BIG-IP Pools list page with the Create button highlighted](./assets/bigip_pool.png)

Fill in the pool-level details:

- **Name (1):** `Scality_Pool`
- **Health Monitor (2):** `Health_HTTP_RING`
- **Load Balancing Method (3):** Least Connections (member)

The `Health_HTTP_RING` monitor created earlier sends a `GET` request to each node's `_/healthcheck/deep` URL and marks the node healthy when it receives `HTTP/1.1 200 OK`.

In the **New Members (4)** section, add the first RING node:

- **Name:** `StorageNode01`
- **Address:** 10.150.91.141
- **Service Port:** 80 (the HTTP S3 data service port for the RING cluster. Specify 443 for HTTPS endpoint)

Click **Add (4)** to save it to the member list. Then enter the next node's details and click **Add (4)** again until all nodes are added. In this example, the RING cluster has three nodes, so add all three nodes in the cluster range 10.150.91.106 through 10.150.91.108 before completing the pool. Once every node appears in the member list, click **Finished (5)**.

![New Pool form with the custom health monitor and RING node members added](./assets/bigip_pool_create_details.png)

The new pool now appears in the Pools list.

![Pools list showing the new ring-cluster pool](./assets/bigip_pool_result.png)

### 2.5 Optimization for S3 Traffic

Tuning the BIG-IP configuration for the specific characteristics of S3 traffic can improve performance and efficiency. This section outlines some of the key optimizations to consider for an S3 workload, including connection limits and TCP profile settings. These optimizations are optional but can help you get the most out of your BIG-IP deployment in front of a Scality RING cluster.

#### 2.5.1 TCP Profiles

A TCP profile controls how the BIG-IP manages TCP connections, such as timers, buffer sizes, and congestion control. The BIG-IP ships with built-in profiles, but the default values are not ideal for RING S3 traffic. For better performance, F5 engineers recommend creating two custom profiles: one for the client side (between the S3 client and the BIG-IP) and one for the server side (between the BIG-IP and the RING nodes). Each side has different needs, so the settings are tuned separately.

The following sections walk through both profiles. Start by navigating to **Local Traffic (1) > Profiles (2) > Protocol (3) > TCP (4)** and clicking **Create (5)**.

![BIG-IP TCP profiles list page with the Create button highlighted](./assets/bigip_tcp_profile_create.png)

##### 2.5.1.1 Client TCP Profile

Set the **Name (1)** to `s3-tcp-custom-client` and set the **Parent Profile (2)** to `s3-tcp`. This creates a new TCP profile that inherits all settings from the `s3-tcp` profile, which is optimized for S3 workloads, and allows you to customize selected values.

![New TCP profile form with the name s3-tcp-custom-client and the s3-tcp parent profile](./assets/bigip_tcp_profile_client_name.png)

In the **Timer Management** section, enable **Minimum RTO (1)** and set it to `200` milliseconds **(2)**. This reduces the minimum retransmission timeout, which can improve performance for S3 workloads that are sensitive to latency.

![Client TCP profile Timer Management section with Minimum RTO set to 200 milliseconds](./assets/bigip_tcp_profile_client_timer.png)

In the **Data Transfer** section, enable **Nagle's Algorithm (1)** and set it to **Auto (2)**. This allows the BIG-IP to decide when to enable or disable Nagle's algorithm based on traffic patterns, which can help optimize throughput for S3 workloads.

![Client TCP profile Data Transfer section with Nagle's Algorithm set to Auto](./assets/bigip_tcp_profile_client_data.png)

In the **Congestion Control** section, enable **Congestion Control (1)** and set it to **CUBIC (2)**. CUBIC is a modern congestion control algorithm that can improve performance in high-bandwidth, high-latency networks, which can benefit S3 traffic.

![Client TCP profile Congestion Control section set to CUBIC](./assets/bigip_tcp_profile_client_congestion.png)

Click the **Finished** button at the bottom of the page to save the client's TCP profile.

##### 2.5.1.2 Server TCP Profile

Create another TCP profile for the server side with the **Name (1)** `s3-tcp-custom-server` and the **Parent Profile (2)** set to `f5-tcp-lan`.

![New TCP profile form with the name s3-tcp-custom-server and the f5-tcp-lan parent profile](./assets/bigip_tcp_profile_server_name.png)

In the **Memory Management** section, set the following values:

- **Auto Receive Window** to **Enabled**
- **Auto Send Buffer** to **Enabled**
- **Proxy Buffer High** to `5200000` bytes
- **Proxy Buffer Low** to `5100000` bytes

![Server TCP profile Memory Management section with the auto window and proxy buffer values set](./assets/bigip_tcp_profile_server_memory.png)

In the **Loss Detection and Recovery** section, turn on **Tail Loss (1)** and set it to **Disabled (2)**.

![Server TCP profile Loss Detection and Recovery section with Tail Loss disabled](./assets/bigip_tcp_profile_server_loss.png)

Click **Finished** at the bottom of the page to save the server's TCP profile.

### 2.6 Virtual Server

A virtual server is the listener that clients connect to. It binds an IP address and port on the BIG-IP to a pool, applies any configured profiles and policies, and forwards traffic to a healthy pool member according to the load balancing method. For this S3 deployment, the virtual server is the single endpoint that S3 clients target instead of talking to individual RING nodes.

As shown in the setup diagram in the [Pool section](#24-pool), the virtual server controls the left segment of traffic, between the S3 client(s) and the BIG-IP. Like the pool, the virtual server can be configured for unencrypted (HTTP) or encrypted (HTTPS) traffic, and this guide provides the steps for both. The two segments are independent: you can, for example, terminate TLS from clients at the BIG-IP while sending either encrypted or unencrypted traffic onward to the RING cluster.

#### 2.6.1 Virtual Server for S3 Client Traffic to BIG-IP

With the pool in place, create the virtual server that exposes it. Navigate to **Local Traffic (1) > Virtual Servers (2)** and click **Create (3)**.

![BIG-IP Virtual Servers list page with the Create button highlighted](./assets/bigip_vs.png)

Fill in the basic details:

- **Name (1):** `RING_Virtual_Server`
- **Source Address (2):** `0.0.0.0/0`
- **Destination Address/Mask (3):** 10.150.92.150 - the virtual IP (VIP) that S3 clients connect to
- **Service Port (4):** 443 (HTTPS)

![New Virtual Server form with name, destination address, and service port filled in](./assets/bigip_vs_create_details_1.png)

Next, configure the virtual server details:

- **1. Protocol (1):** Set to **TCP**
- **2. Protocol Profile (Client) (2):** Set to the custom client TCP profile you created earlier (`s3-tcp-custom-client`)
- **3. Protocol Profile (Server) (3):** Set to the custom server TCP profile you created earlier (`s3-tcp-custom-server`)
- **4. HTTP Client Profile (4):** Set to **HTTP**
- **5. SSL Profile (Client) (5):** Set to the default client SSL profile (`clientssl`). Skip this if you want to use unencrypted HTTP between S3 clients and the BIG-IP.
- **6. SSL Profile (Server) (6):** Set to the default server SSL profile (`serverssl`) if your RING cluster serves encrypted S3 traffic. Skip if serves only HTTP on port 80.
- **7. Source Address Translation (7):** Set to **Auto Map** to allow the BIG-IP to manage source address translation for return traffic.

![Virtual Server form with Source Address Translation set to Auto Map](./assets/bigip_vs_create_details_2.png)

In the **Resources** section, select the `Scality_Pool` pool from the **Default Pool (1)** dropdown to bind the virtual server to the backend nodes.

Click **Finished (2)** to save the virtual server.
![Virtual Server Resources section with ring-cluster selected as the Default Pool](./assets/bigip_vs_create_details_3.png)

The virtual server now appears in the list. A green circle next to its name indicates that the virtual server is up and at least one pool member is passing health checks.

![Virtual Servers list showing ring-cluster-vs with a green available status](./assets/bigip_vs_result.png)

### 2.7 Test the Virtual Server

With the virtual server up and at least one healthy pool member, validate that S3 traffic flows end to end through the BIG-IP.

#### 2.7.1 Configure the RING Cluster

Before you test traffic, create an S3 bucket, a user, and an access key in the Scality RING cluster. After the login click the **S3 Services**

##### 2.7.1.1 Create Account

![RING Dashboard](./assets/scality_overview.png)

The S3 Services page allows access to Scality UI services. To create a user, bucket and access key/secret the `S3 Console` is used. Click the **Log in (1)** button.
![S3 Services](./assets/scality_s3services.png)

To create a test user, in the appeared `Accounts` page click **Create accoung (1)** button.
![Accounts Dashboard](./assets/scality_accounts.png)

The popup with user info will appear. Specify the user infomation:

- **Account name (1):** warpuser
- **Email address (2):** warpuser@test.domain
- **Password (3):** specify a password here
- **Password confirmation (4):** confirm the password specified before

Remember the password and account name. We will need them later.
![Scality Account Create](./assets/scality_create_acc.png)

Click the **Submit (5)** button to create a user.

The successfully created account will be added to the accounts list:
![Scality Accounts List with a new account](./assets/scality_create_acc_result.png)

As a next step, to create resources under the new account, the current account needs to be logged out. Click **Admin account (1)** profile at the top right part of the screen. Select **Log out (2)**.
![Logout admin](./assets/scality_accounts_logout.png)

In login form enter the new created user credentials:

- **USERNAME: (1)** warpuser
- **PASSWORD: (2)** password spicified during account creation

Click **Submit (3)** button.

![warpuser login](./assets/scality_warpuser_login.png)

##### 2.7.1.2 Create S3 User

After login to the account you can create an S3 user. Click **+ (1)** button.
![Add S3 User](./assets/scality_add_s3_user.png)

In the appeared dialog:

- **Username: (1)** warp-s3-user
- Select **FullAccessGroup (2)**

![Create S3 user](./assets/scality_warp_s3_user_create.png)

Click **Submit (3)** button.

The created user will be shown in the users list. Select the **Key (1)** button in the actions list.
![Create S3 user result](./assets/scality_warp_s3_user_create_res.png)

In order to connect to the S3 environment the user needs to have an`Access Key` and a `Secret`. Click the **+ Generate a new key (1)** button
![Create S3 user result](./assets/scality_warp_s3_user_access_keys.png)

The new `Access Key` will be generated. Save the **AccessKey (1)** and click **Show (2)** button near the **SecretAccessKey**. Save the `Secret` too. You will need them further. Make sure to not share the credentials with anyone.

![S3 user access credentials](./assets/scality_warp_s3_user_access_keys_res.png)

Click **Close (3)** button to close the current dialog.

##### 2.7.1.3 Create S3 Bucket

In Scality Ring administration console select **S3 Services (1)** and click **Login (2)** to `S3 Browser`.
![Scality Ring S3 Browser Login](./assets/scality_ring_s3browser.png)

Then use saved `Access key` and `Secret key`to login. Enter access key to **Access Key (1)** field and **Secret key (2)** field.

![Scality Ring S3 Browser Login](./assets/scality_s3browser_login.png)

Click **LOGIN (3)** button.

Click **CREATE BUCKET (1)** button.
![Scality buckets](./assets/scality_s3browser_buckets.png)

The create bucket dialot will appear. Specify `warp-connectivity-test` for **Bucket Name (1)** and click **CREATE (2)** button.
![Scality create bucket](./assets/scality_bucket_create.png)

Bucket will be created:
![Scality create bucket result](./assets/scality_bucket_create_res.png)

#### 2.7.2 Use Warp to Test Connectivity to the RING S3 Cluster Through the BIG-IP Virtual Server

Warp is an S3 benchmarking tool. It drives a representative S3 workload (mixed PUT, GET, DELETE, and STAT operations) against an endpoint and reports throughput, latency, and error counts. Pointing Warp at the BIG-IP virtual server address, rather than at an individual RING node, confirms that traffic flows end to end through the load balancer and gives a baseline measurement of the cluster's performance behind the BIG-IP.

Download and install the Warp CLI from the official download page:

[https://www.min.io/download/minio-warp](https://www.min.io/download/minio-warp)

Configure Warp with the BIG-IP virtual server address as the S3 endpoint, along with the access key ID and secret access key generated on the RING cluster, and run a short mixed-workload test:

```bash
export WARP_HOST=10.150.92.150:443               # the BIG-IP virtual server address and port
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


Requests: 11 | Objects: 11 | Bytes: 11264 | Errors: 0
```

An error count of zero confirms that every request reached the RING cluster and returned a successful response through the virtual server.

### 2.8 Automation with Ansible

The manual steps above can be reproduced declaratively with Ansible. Playbooks, roles, and inventory live in [automation/](./automation/). See [automation/README.md](./automation/README.md) for installation, credentials, and usage.
