To configure **NetApp Onboard DNS Load Balancing** with **Infoblox** as the DNS forwarder, you need to set up NetApp’s **onboard DNS** to handle load balancing across its LIFs and use Infoblox as the external DNS server for the environment. Infoblox will handle external DNS requests and forward them to the NetApp cluster's onboard DNS for load balancing.

Here’s the step-by-step guide for configuring **NetApp Onboard DNS Load Balancing** with **Infoblox** as the DNS forwarder:

### Steps to Configure NetApp Onboard DNS Load Balancing with Infoblox

#### 1. **Ensure NetApp Cluster and LIF Configuration**
Ensure that your NetApp cluster is set up with multiple **Logical Interfaces (LIFs)** on a Storage Virtual Machine (SVM), each having its own IP address and data protocol (NFS, CIFS, etc.). 

Check the existing LIFs using:

```bash
network interface show
```

If you don’t have multiple LIFs set up, create them for each node in the NetApp cluster using:

```bash
network interface create -vserver <SVM_name> -lif <LIF_name> -role data -data-protocol nfs,cifs -home-node <node_name> -home-port <port_name> -address <IP_address>
```

#### 2. **Enable Onboard DNS Load Balancing on the NetApp Cluster**

You will configure the NetApp cluster to act as an onboard DNS server with load balancing enabled. Here are the steps:

##### a. **Enable the DNS Load Balancing Service**

To enable the onboard DNS service on your vServer, run the following:

```bash
vserver services name-service dns create -vserver <SVM_name> -domains <domain_name> -name-servers <Infoblox_DNS_IP> -load-balancing enabled
```

Example:

```bash
vserver services name-service dns create -vserver my_svm -domains example.com -name-servers 192.168.100.10 -load-balancing enabled
```

- **Infoblox_DNS_IP** is the IP address of your Infoblox DNS server. 
- The `-load-balancing enabled` flag ensures that NetApp will use its internal DNS for load balancing across LIFs.

##### b. **Map Hostnames to LIF IP Addresses**

Now, configure the hostname that clients will use to access the storage resources, and map it to the LIF IPs.

```bash
vserver services name-service dns hosts create -vserver <SVM_name> -hostname <hostname> -addresses <comma_separated_IPs>
```

For example:

```bash
vserver services name-service dns hosts create -vserver my_svm -hostname myfileserver.example.com -addresses 10.10.10.1,10.10.10.2,10.10.10.3
```

This command maps the hostname `myfileserver.example.com` to multiple IPs (`10.10.10.1`, `10.10.10.2`, and `10.10.10.3`) for load balancing.

#### 3. **Configure Infoblox DNS to Forward to NetApp’s Onboard DNS**

You need to configure **Infoblox** to forward DNS queries for specific domains or hostnames to the NetApp cluster’s onboard DNS service.

##### a. **Log in to Infoblox**:

1. Open the Infoblox web interface and log in.
2. Navigate to the **Data Management** tab.
3. Go to **DNS** > **Zones**.

##### b. **Create a New DNS Zone** (if needed):

- Add a forward zone or edit an existing zone for your NetApp-related domain (e.g., `example.com`).
- In the **Forwarding Servers** section, add the IP address of the NetApp cluster (or the IP addresses of its LIFs if necessary).

For example, if your NetApp onboard DNS is accessible via `10.10.10.1`, set this as a forwarding server.

- Specify the **domain** or **subdomain** to forward (e.g., `myfileserver.example.com` or `*.example.com`), ensuring that all queries for this domain are forwarded to the NetApp DNS load balancer.

##### c. **Configure DNS Forwarders**:

1. In Infoblox, go to **Grid** > **DNS** > **Members** > **Edit Member**.
2. Scroll to **DNS Forwarding Member** settings.
3. Add the NetApp onboard DNS IP (the LIFs or management IP) as a forwarder.

#### 4. **Testing DNS Forwarding and Load Balancing**

After configuring the onboard DNS load balancing and Infoblox forwarding, you can test to ensure the integration works.

##### a. **Query from a Client Machine**

Run the following command on a client machine to check DNS resolution:

```bash
nslookup myfileserver.example.com <Infoblox_DNS_IP>
```

You should see responses with different IPs for each query as DNS round-robin load balancing distributes traffic across the LIFs.

##### b. **Verify DNS Configuration on NetApp**

Run the following command to confirm the onboard DNS and load-balancing configuration on the NetApp:

```bash
vserver services name-service dns show
```

It should display your DNS settings, showing the onboard DNS server, domains, and the load-balancing status.

##### c. **Check Load Balancing Behavior**

Verify that the client is receiving different IP addresses in each query:

```bash
nslookup myfileserver.example.com
```

The IP addresses returned should alternate between the LIF IPs based on the load balancing settings.

#### 5. **Monitor and Tune Load Balancing**

You can fine-tune the load balancing behavior by adjusting settings on the NetApp cluster or Infoblox, including:

- **TTL settings**: Adjust the Time-to-Live (TTL) for DNS records on the Infoblox side to control how long clients cache IP addresses before requesting them again from the DNS server.
- **Load Balancing Policies**: In NetApp, you can configure different load balancing policies such as **round-robin**, **IP hash**, or **session stickiness**:

```bash
vserver services name-service dns modify -vserver <SVM_name> -load-balancing-policy round-robin
```

This can improve performance based on your network traffic and workload.

### Summary

1. **Set up NetApp Onboard DNS Load Balancing** by enabling it on the vServer and configuring hostnames and IP addresses.
2. **Configure Infoblox** as the external DNS server, with forwarding rules directing queries for your domain to NetApp’s onboard DNS.
3. **Test DNS resolution and load balancing** by running queries and ensuring that DNS returns balanced IPs across multiple queries.
4. **Fine-tune the system** by adjusting load balancing policies and TTL values.

This setup leverages the Infoblox DNS server to forward client DNS requests to NetApp’s onboard DNS, which handles intelligent load balancing across the available LIFs.
