# Deployment Patterns

The following sections provide high level information on
the recommended deployment pattern. 


## Introduction

You can run multiple nodes of WSO2 Identity Server in a cluster mode to achieve two requirements.

1. Handle requests seamlessly: If one node becomes unavailable or is experiencing high traffic, another node will 
seamlessly handle the requests. 
2. Balancing traffic handling: Multiple nodes can handle the traffic together so that cluster throughput is higher 
than the throughput of a single node.

<!--For complete information on clustering concepts, see [Clustering Overview](../../../deploy/clustering-overview). -->
The following sections guide you through setting up the deployment pattern, which is an HA Clustered Deployment 
of two WSO2 Identity Server nodes. 

---

## Deployment prerequisites

As a first step in planning your deployment, ensure that you have the
necessary system requirements and a compatible environment.

## System requirements

<table style="width:100%;">
<colgroup>
<col style="width: 11%" />
<col style="width: 88%" />
</colgroup>
<tbody>
<tr class="odd">
<td></td>
<td><ul>
<li>4 vCPUs</li>
<li>4 GB RAM</li>
<li>10 GB Disk Space</li>
</ul>
</td>
</tr>
</tbody>
</table>

The above recommendations can change based on the expected concurrency and performance.

!!! note
    For more information on prerequisites, see [Installation Prerequisites](../../../deploy/get-started/install).

---

## Environment compatibility

<table>
<tbody>
<tr class="odd">
<th>Operating systems</th>
<td><p>For information on tested operating systems, see <a href="../../../deploy/environment-compatibility/#tested-operating-systems-and-jdks">Tested Operating Systems and JDKs</a> 
.</p></td>
</tr>
<tr class="even">
<th>RDBMS</th>
<td><p>For information on tested DBMSs, see <a href="../../../deploy/environment-compatibility/#tested-dbmss">Tested DBMSs</a> .</p></td>
</tr>
<tr class="odd">
<th>Directory services</th>
<td><p>Supports Directory Services implementing following LDAP Protocols:</p>
<ul>
<li>LDAP v2</li>
<li>LDAP v3</li>
</ul>
<p>For information on tested LDAPs, see <a href="../../../deploy/environment-compatibility/#tested-ldaps">Tested LDAPs</a> .</p></td>
</tr>
<tr class="even">
<th>Java</th>
<td>Oracle JDK 1.8</td>
</tr>
<tr class="odd">
<th>Web browsers</th>
<td><p>For more information on tested web browsers, see <a href="../../../deploy/environment-compatibility/#tested-web-browsers">Tested Web Browsers</a> .</p></td>
</tr>
<!--<tr class="even">
<th>Load balancers</th>
<td><p>For more information about load balancers, see <a href="../../../deploy/load-balancing/#types-of-load-balancers">Configuring the load balancer</a> .</p></td>
</tr>-->
</tbody>
</table>

---

## Configure databases

In a clustered deployment, all WSO2 Identity Server nodes are pointed to the same databases to ensure the integrity 
of the data. Also, you can configure multiple logical databases if you require to keep your data logically separated 
in the environment. The following tutorial demonstrates deployment with an identity database (**IDENTITY_DB**) and a user 
database (**UM_DB**).

!!! note
    Alternatively, you can create more databases for each type of data to separate the data logically.  
    This will not make a difference in performance and is not mandatory. Separating databases logically may 
    sometimes help to have a different backup and scaling strategy when the deployment is large and complex.
    If you do wish to separate the data logically into separate databases, 
    see [Set Up Separate Databases for Clustering](../../../deploy/set-up-separate-databases-for-clustering).

!!! tip
    If you have configured the shared database correctly, the `deployment.toml` file in 
    `<IS_HOME>/repository/conf` directory should have the following configurations. 
    
    Following is a sample configuration. Therefore parameter values might be different.
    
    ```toml
    [database.identity_db]
    type = "mysql"
    hostname = "localhost"
    name = "regdb"
    username = "regadmin"
    password = "regadmin"
    port = "3306"
    
    [database.shared_db]
    type = "mysql"
    hostname = "localhost"
    name = "regdb"
    username = "regadmin"
    password = "regadmin"
    port = "3306"
    ```

The following diagram is a high-level component diagram showing how the system would look like when two databases 
are used.

![Component diagram](../../assets/img/deploy/component-diagram.png)

!!! note
    For instructions on how to configure the data sources for other databases and 
    more information related to databases, 
    see [Work with Databases](../../../deploy/work-with-databases)

---

## Mount the shared registry

WSO2 Identity Server comprises three different registry repositories.

1. **Local Repository**: Stores configuration and runtime data that is local to the server

2. **Configuration Repository**: Stores product-specific configurations 

3. **Governance Repository**: This stores configuration and data that are shared across the whole platform. This typically 
includes services, service descriptions, endpoints, or data sources.


<!--!!! info
    For more information about the registry, 
    see [Work with the Registry](TO-DO:../../guides/working-with-the-registry).-->

In this cluster setup, we use the default h2 database as the local registry in each node individually, and the 
governance and configuration registries should be mounted to share across all nodes. In WSO2 Identity Server, config and governance registries are mounted by default.

!!! note
    The production recommendation is to set the `<versionResourcesOnChange>` property in the `registry.xml` 
    file to false. This is because the automatic versioning of resources can be an extremely expensive operation. 

    ```
    <versionResourcesOnChange>false</versionResourcesOnChange>
    ```

To make sure the configurations were applied correctly,

<ol>
    <li>Log in to the WSO2 Identity Server Management Console (<code>https://&lt;IS_HOST&gt;:&lt;PORT&gt;/carbon</code>).</li>
    <li>
        <p>Navigate to <b>Home > Registry > Browse</b>.</p>
        <p><img src="../../assets/img/deploy/registry-browser-2.png" alt="Registry browser"></p>
        <p>Note that the governance collection is shown with the symlink icon.</p>
    </li>    
</ol>

---

## Clustering-related configurations

WSO2 supports the following membership schemes for clustering

-   Well Known Address (WKA)
-   Multicast membership 
-   AWS membership 
-   AWS ECS membership 
-   Kubernetes membership

The following configurations need to be done in both the WSO2 Identity Server nodes in order to enable clustering between 
them.

1. Enable clustering on node 1 and node 2 by setting the membership
   scheme that fits your deployment by editing the
   `<IS_HOME>/repository/conf/deployment.toml` file.

    !!! info
        The simplest is the well-known address (WKA) based clustering method. It only suites where all the nodes are 
        deployed on machines having static IP addresses. <!--For more information, see [About Membership Schemes](../../../deploy/clustering-overview/#about-membership-schemes).-->
        Configurations for each membership scheme are listed below.
        
        ??? tip "Click to see the instructions for WKA scheme"            
            Edit the `<IS_HOME>/repository/conf/deployment.toml` file to add following configurations.
            Configure the `localMemberHost` and `localMemberPort` entries. Add the IP of the editing node itself.                    
                    ```
                    [clustering]
                    membership_scheme = "wka"
                    local_member_host = "192.168.2.1"
                    local_member_port = "4000"
                    members = ["192.168.2.1:4000", "192.168.2.2:4001"]
                    ```                    
            Under the `members` section, add the `hostName` and `port` for each WKA member. As we have only two nodes 
            in our sample cluster configuration, we will configure both nodes as WKA nodes.         
            
            You can also use IP address ranges for the `hostName`. For example, `192.168.1.2-10`. This should ensure 
            that the cluster eventually recovers after failures. One shortcoming of doing this is that you can define 
            a range only for the last portion of the IP address. You should also keep in mind that the smaller 
            the range, the faster it discovers members since each node has to scan a lesser 
            number of potential members. 
            
        ??? tip "Click to see the instructions for AWS ECS membership scheme"  
                      
            1. Create a working AWS ECS Cluster. Note the following when creating a cluster.
                -   Note the `name` and `VPC CIDR block` of the cluster as you will require them later for configurations.
                -   Ensure that the `Container instance IAM role` that you assign to the ECS cluster has the following permission policy attached. 
                        ```
                        { "Version": "2012-10-17", 
                             "Statement":
                             [
                             {
                                 "Effect": "Allow",
                                 "Action":
                                     [
                                     "ec2:DescribeAvailabilityZones",
                                     "ec2:DescribeInstances"
                                     ],
                                     "Resource": [ "*" ]
                             }
                             ]
                        }
    
                        ```
                -   Make sure that the security group of the cluster instances has an inbound rule to allow incoming 
                traffic on the Hazelcast default port range `(5701 - 5708)`. It is advised to restrict the access to 
                instances in the same security group for this inbound rule. 
            
            2. Create a `deployment.toml` file in a preferred directory to add following configurations.
            Configure the following entries.                    
                    ```
                    [clustering]
                    membership_scheme = "aws-ecs"
                    
                    [clustering.properties]
                    region = "us-east-1"
                    clusterName = "ECS-IS-CLUSTER"
                    vpcCidrBlock = "10.0.*.*"
                    ```                    
            Under the `clustering.properties` section, set the `region`, `clusterName`, and `vpcCidrBlock` based on 
            the AWS ECS cluster that you created in the previous step.       

            !!! note
                Once all the configurations are complete, build a docker image including the configurations. You can 
                consume this docker image to 
                create a `Task Definition` and run a new `Service` or a `Task` 
                on the `AWS ECS cluster` that you created.
            
        
2. Configure caching.

    !!! note
        Distributed caching is disabled and it is not recommended 
        to use this due to many practical issues that are related to configuring and running distributed 
        caching properly. WSO2 Identity Server employs **Hazelcast** as the primary method of implementing 
        cluster messages while using distributed caching in a simple setup.

    ??? info "About Caching"
        <ul>
            <li><b>Why caching</b></br>Caching is an additional layer on top of the databases. It enables to keep 
            the recently used data that are fetched from the database in local memory, so that for subsequent 
            data requests instead of fetching from the database the data can be served from the local memory. 
            Caching has certain advantages and disadvantages that you need to evaluate when deciding on your 
            caching strategy.</li>
            <li><b>Advantages</b>
                <ul>
                    <li>The load on the underlying database or LDAP is reduced as data is served from already 
                    fetched data in memory.</li>
                    <li>Improved performance due to the reduced number of database calls for repetitive 
                    data fetching</li>
                </ul>
            </li>
            <li><b>Disadvantages</b>
                <ul>
                    <li>Coherency problems may occur when the data change is not immediately reflected on 
                    cached data if one node or an external system updates the database.</li>
                    <li>Data in memory can become stale yet be served, e.g., serving data from memory while 
                    its corresponding record in the database is deleted.</li>
                </ul>
            </li>
        </ul>

    ??? tip "Caching in WSO2 Identity Server"
        Historically WSO2 Identity Server used distributed caching to utilize the above-mentioned advantages 
        as well as to minimize the coherence problem. However, in newer deployment patterns where the network 
        is not tightly controlled, distributed caching fails in unexpected ways. Hence, we **no longer recommend 
        using distributed caching**. Instead, it is **recommended to have local caches** (if required) and 
        **cache invalidation messages** (if required) by considering the information given below.
        
        **The ForceLocalCache property**

        When Hazelcast clustering is enabled, certain caches act as distributed caches. The `force_local_cache` property in the `<IS_HOME>/repository/conf/deployment.toml` directory is there to mark that all the caches should act like local caches even in a clustered setup. (This is by default set to `true`).
        ```
        [server]
        force_local_cache = true
        ```
        Cache invalidation uses Hazelcast messaging to distribute the invalidation message over the cluster 
        and invalidate the caches properly. This is used to minimize the coherence problem in a multi-node setup.
        
        **Typical clustered deployment cache scenarios**

        <table>
            <thead>
                <tr>
                    <th>Scenario</th>
                    <th>Local<br>Caching</th>
                    <th>Distributed<br>Caching</th>
                    <th>Hazelcast<br>Clustering</th>
                    <th>Distributed<br>Invalidation</th>
                    <th>Description</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td>1. All caches are local with distributed cache invalidation</td>
                    <td>Enabled</td>
                    <td>Not<br> Applicable</td>
                    <td>Enabled</td>
                    <td>Enabled</td>
                    <td>
                        <ul>
                            <li>This is the <b>recommended approach</b>.</li>
                            <li>Hazelcast messaging invalidates the caches.</li>
                        </ul>
                    </td>
                </tr>
                <tr>
                    <td>2. All caches are local without distributed cache invalidation</td>
                    <td>Enabled</td>
                    <td>Not<br> Applicable</td>
                    <td>Disabled</td>
                    <td>Disabled</td>
                    <td>
                        <ul>
                            <li>Invalidation clears only the caches in specific nodes. Other caches are 
                            cleared at cache expiration.</li>
                            <li>Hazelcast communication is not used.</li>
                            <li>As the decisions take time to propagate over nodes (default cache 
                            timeout is 15 minutes), there is a security risk in this method. To reduce 
                            the risk, reduce the default cache timeout period. To learn how to reduce the 
                            default cache timeout period, 
                            see [Configuring Cache Layers - timeout](../../../deploy/performance/configure-cache-layers#timeout).</li>
                        </ul>
                    </td>
                </tr>
                <tr>
                    <td>3. No caching</td>
                    <td>Disabled</td>
                    <td>Disabled</td>
                    <td>Disabled</td>
                    <td>Disabled</td>
                    <td>
                        <ul>
                            <li>The data are directly acquired from the database.</li>
                            <li>Eliminates the security risks caused due to not having cache invalidation.</li>
                            <li>This method will create a performance degradation due to the lack of 
                            caching.</li>
                        </ul>
                    </td>
                </tr>
                <tr>
                    <td>4. Certain caches are disabled while the remaining are local</td>
                    <td>Enabled<br>for the <br> available local caches</td>
                    <td>Not<br> Applicable</td>
                    <td>Enabled</td>
                    <td>Enabled</td>
                    <td>
                        <ul>
                            <li>To reduce the security risk created in the second scenario and to 
                            improve performance in comparison with the third scenario, disable the 
                            security-related caches and sustain the performance-related caches as 
                            local caches.</li>
                            <li>This requires identification of these caches depending on the use case.</li>
                        </ul>
                    </td>
                </tr>
                <tr>
                    <td>5. Distributed caching enabled</td>
                    <td>Disabled <br> - the <br> `force_local_cache` is<br> set to `false`.</td>
                    <td>Enabled</td>
                    <td>Enabled</td>
                    <td>Not<br> Applicable</td>
                    <td>
                        <ul>
                            <li>This scenario is only recommended if the network has tight tolerance 
                            where the network infrastructure is capable of handling high bandwidth with 
                            very low latency.</li>
                            <li>Typically this applies only when you deploy <b>all the nodes in a single 
                            server rack having fiber-optic cables</b>. In any other environments, this 
                            implementation will cause cache losses. Thus, this implementation is <b>not 
                            recommended for general use</b>.
                        </ul>
                    </td>
                </tr>
            </tbody>
        </table>
        </li>
    </ul>

3. Go to the `<IS_HOME>/repository/conf/deployment.toml` file and add the proxy port as `443`. 
The port 443 is the Load Balancer frontend port.

    !!! abstract ""
        **Example**
        ```
        [transport.http.properties]
        proxyPort = 80
        [transport.https.properties]
        proxyPort = 443 
        ```

4. You may change the `<IS_HOME>/repository/conf/deployment.toml` file to access the servers using a 
hostname instead of the raw IP. This hostname is how the external applications try to look up WSO2 
Identity Server endpoints. The `hostName` should be resolved to the Load Balancer front-end IP address.

    ```
    [server]
    hostname = "wso2.is"
    ```

    !!! note 
        This hostname is used by the WSO2 Identity Server cluster and therefore it must be specified in 
        the `/etc/hosts` file in each node so that internal calls will not be routed through the Load Balancer.
        
        Example: 
        `192.168.2.1   wso2.is`

---

## Enable artifact synchronization

To enable synchronization for runtime artifacts, you must have a shared file system. You can use one of the following
 depending on your environment.
 
   -   **Network File System (NFS)**: This is one of the most commonly known shared file systems and can be used 
   in a Linux environment.
   -   **Server Message Block (SMB) file system**: This can be used in a Windows environment.
   -   **Amazon EFS**: This can be used in an AWS environment.

Once you have chosen a file system, 

1. Mount it in the nodes that are participating in the cluster.
2. Create two directories called `Deployment` and `Tenants` in the shared file system.
3. Create a symlink from the `<IS_HOME>/repository/deployment` path to the `Deployment` directory of the shared 
file system that you created in the previous step.
4. Create a symlink from the `<IS_HOME>/repository/tenants` path to the `Tenants` directory of the shared file 
system that you created.

!!! note
    Instead of mounting the file system directly to the `<IS_HOME>/repository/deployment` and
     `<IS_HOME>/repository/tenants` paths, a symlink is created. Otherwise, if you delete the product to redeploy it, the file system would get mounted to a non-existing path.
 
---

## Front with a load balancer

In order to access the two WSO2 Identity Server nodes, you need to front the system with a load balancer. 
You can use any load balancer that is available to your system. 

!!! info
    To learn how to front the 2 node cluster with an Nginx load balance, 
    see [Set up WSO2 clusters with Nginx](../../../deploy/front-with-the-nginx-load-balancer). 
    If you’ve changed the hostnames of nodes, make sure to keep that in mind when you’re configuring your 
    load balancer.

---

## Run the cluster

1. Start the load balancer and WSO2 Identity Server nodes.
2. Access the WSO2 Identity Server Management Console (`https://<IS_HOST>:<PORT>/carbon`). 
    Traffic will be served by one of the nodes in the cluster, depending on your load balancer.
3. Shut down the cluster node 1 and observe that the traffic is served by node 2. 
3. Start node 1 and shut down node 2. Note that traffic will be served by node 1.
