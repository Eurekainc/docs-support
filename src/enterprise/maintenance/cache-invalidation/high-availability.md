---
summary: Learn how to configure a RabbitMQ cluster and how you can make OutSystems cache invalidation service fault-tolerant.
tags: version-11; support-Installation_Configuration
---

<pre class="script">
template('OutSystems/OSVersionIndicator');
</pre>

# OutSystems cache invalidation with high-availability

RabbitMQ can be configured to work as a **cluster**, offering benefits like fault-tolerance, high availability and data replication. If your application requires that the cache invalidation service is always available, you should use a RabbitMQ cluster.

In this guide, we will help you set up a basic RabbitMQ cluster that can be used to ensure high availability of the cache invalidation mechanism used by OutSystems applications. We will also show you how to make OutSystems cache invalidation tolerant to the failure of a RabbitMQ single node by configuring a TCP load balancer.

Check RabbitMQ's official documentation for in-depth information on [RabbitMQ clusters](<https://www.rabbitmq.com/clustering.html>).

## Create a distributed RabbitMQ cluster

In this example we will create a simple RabbitMQ cluster with two nodes running on separate servers, "ServerA" and "ServerB".

### Step 1. Install RabbitMQ on each machine

This step is beyond the scope of this guide; check the official documentation for more information on [how to install and configure RabbitMQ](<https://www.rabbitmq.com/install-windows.html>).

After installing RabbitMQ on two separate machines, each node will be completely unaware of the existence of the other:

![](<images/cluster-two-instances.png>)

In the picture above we assume that, after installing both servers, the OutSystems platform was configured with the address of "ServerA" for its cache invalidation service.

### Step 2. Aggregate the two nodes into a single cluster

<div class="info" markdown="1">

In the commands presented next, the address `rabbit@ServerB` means «the RabbitMQ broker named `rabbit` running on "ServerB"». The name of the RabbitMQ broker is `rabbit` by default, though this name can be configured.

</div>

To aggregate both nodes into a single cluster, do the following:

1. In "ServerA", open a command-line in the `<RabbitMQ installation folder>\sbin` folder.  
If you installed RabbitMQ via Configuration Tool, the install folder is located in `<OutSystems install folder>\Platform Server\thirdparty\RabbitMQ Server\rabbitmq_server-<version>`.

1. Run the following commands, replacing `ServerB` with the correct hostname:

        rabbitmqctl -n rabbit@ServerB stop_app
        rabbitmqctl -n rabbit@ServerB join_cluster rabbit
        rabbitmqctl -n rabbit@ServerB start_app

    _Note:_ The hostname is **case-sensitive**.

    These commands do the following:  
    — Stop the RabbitMQ broker named `rabbit` in "ServerB";  
    — Make the RabbitMQ broker in "ServerB" join the RabbitMQ broker in "ServerA", also named `rabbit`, in a cluster;  
    — Restart the RabbitMQ broker in "ServerB".

After running the commands you should now have RabbitMQ cluster ready to be used:

![](<images/cluster-created.png>)

<div class="info" markdown="1">

By default, data is shared among RabbitMQ nodes as if a single RabbitMQ instance is running. To customize how data and queues are replicated across nodes, check the links in the "See Also" section.

</div>

## Make OutSystems cache invalidation fault-tolerant

In the previous section you learned how to configure a RabbitMQ cluster with two nodes, "ServerA" and "ServerB".

However, OutSystems modules continue to communicate with a single RabbitMQ node, "ServerA". If the communication with this node fails for some reason, the cache invalidation also stops working.

To make OutSystems modules fault-tolerant to a RabbitMQ node failure, you must configure a **TCP load balancer** that manages all the connections made to the configured cache invalidation service address. The load balancer acts as an intermediary between OutSystems modules and the RabbitMQ cluster.

_Note:_ There are alternatives to the load balancer solution, such as having a dynamic DNS configuration. This kind of configuration is beyond the scope of this guide.

### Example: Set up NGINX as a load balancer

You can use any load balancer that supports TCP, such as HAProxy or Microsoft Network Balancer. In this example, we use NGINX as a load balancer for our cluster.

Following our example, create a simple load balancer with NGINX by doing the following:

1\. Install NGINX on the server that acts as a load balancer.

2\. Replace the contents of the NGINX configuration file (stored in `<NGINX installation folder>\conf\nginx.conf`) with the contents provided below:

```
events {
    worker_connections 1024;
}

stream {
    upstream rabbits {
        server <ServerA>:5672;
        server <ServerB>:5672;
    }

    server {
        listen        5672;
        proxy_pass    rabbits;
        proxy_timeout 120s;
        proxy_connect_timeout 5s;
    }
}
```

Replace `<ServerA>` and `<ServerB>` with the appropriate fully-qualified hostname of each server. Check the RabbitMQ's official documentation to learn more on [configuring how NGINX handles load balancing](<https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/>).

3\. Start NGINX by going to the installation folder and running the following command:

```
start nginx
```

4\. Open the Configuration Tool and update the cache invalidation service configuration to use the NGINX load balancer you just created by using its address as the service address.

5\. Re-apply settings to every module in your factory.

After following the previous steps, you now have a load balancer distributing connections made by OutSystems modules to each node in the RabbitMQ cluster:

![](<images/cluster-load-balancer.png>)

<div class="info" markdown="1">

**High-availability load balancer**

This guide doesn't cover how to configure a highly available load balancer. To learn more about this topic, check [An Introduction to HAProxy and Load Balancing Concepts](<https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts#high-availability>) ("High Availability" section) by DigitalOcean. Although the article is focused on HAProxy, the same principles apply to any kind of load balancer.

</div>