---
title: Migrate on-premises Apache Hadoop clusters to Azure HDInsight - infrastructure best practices
description: Learn infrastructure best practices for migrating on-premises Hadoop clusters to Azure HDInsight.
services: hdinsight
author: hrasheed-msft
ms.reviewer: ashishth
ms.service: hdinsight
ms.custom: hdinsightactive
ms.topic: conceptual
ms.date: 10/25/2018
ms.author: hrasheed
---
# Migrate on-premises Apache Hadoop clusters to Azure HDInsight - infrastructure best practices

This article gives recommendations for managing the infrastructure of Azure HDInsight clusters. It's part of a series that provides best practices to assist with migrating on-premises Apache Hadoop systems to Azure HDInsight.

## Plan well for the capacity needed for HDInsight clusters

The key choices to make for HDInsight cluster capacity planning are the following:

- **Choose the region** - The Azure region determines where the cluster is physically provisioned. To minimize the latency of reads and writes, the cluster should be in the same Region as the data.
- **Choose storage location and size** - The default storage must be in the same Region as the cluster. For a 48-node cluster, it is recommended to have 4 to 8 storage accounts. Although there may already be sufficient total storage, each storage account provides additional networking bandwidth for the compute nodes. When there are multiple storage accounts, use a random name for each storage account, without a prefix. The purpose of random naming is reducing the chance of storage bottlenecks (throttling) or common-mode failures across all accounts. For better performance, use only one container per storage account.
- **Choose the VM size and type (now supports the G-series)** - Each cluster type has a set of node types, and each node type has specific options for their VM size and type. The VM size and type is determined by CPU processing power, RAM size, and network latency. A simulated workload can be used to determine the optimal VM size and type for each node types.
- **Choose the number of worker nodes** - The initial number of worker nodes can be determined using the simulated workloads. The cluster can be scaled later by adding more worker nodes to meet peak load demands. The cluster can later be scaled back when the additional worker nodes are not required.

For more information, see the article [Capacity planning for HDInsight clusters](../hdinsight-capacity-planning.md)

## Use the recommended virtual machine types for cluster nodes

See [Default node configuration and virtual machine sizes for clusters](../hdinsight-component-versioning.md#default-node-configuration-and-virtual-machine-sizes-for-clusters) for recommended virtual machine types for each type of HDInsight cluster.

## Check the availability of Hadoop components in HDInsight

Each HDInsight version is a cloud distribution of a version of Hortonworks Data Platform (HDP) and consists of a set of Hadoop eco-system components. See [HDInsight Component Versioning](../hdinsight-component-versioning.md) for details on all HDInsight components and their current versions.

You can also use Ambari UI or Ambari REST API to check the Hadoop components and versions in HDInsight.

Applications or components that were available in on-premises clusters but aren't part of the HDInsight clusters can be added on an Edge Node or on a VM in the same VNet as the HDInsight cluster. A third-party Hadoop application that isn't available on Azure HDInsight can be installed using the "Applications" option in HDInsight cluster. Custom Hadoop applications can be installed on HDInsight cluster using "script actions". The following table lists some of the common applications and their HDInsight integration options:

|**Application**|**Integration**
|---|---|
|Airflow|IaaS or HDInsight Edge node
|Alluxio|IaaS  
|Arcadia|IaaS 
|Atlas|None (Only HDP)
|Datameer|HDInsight Edge node
|Datastax (Cassandra)|IaaS (CosmosDB an alternative on Azure)
|DataTorrent|IaaS 
|Drill|IaaS 
|Ignite|IaaS
|Jethro|IaaS 
|Mapador|IaaS 
|Mongo|IaaS (CosmosDB an alternative on Azure)
|NiFi|IaaS 
|Presto|IaaS or HDInsight Edge node
|Python 2|PaaS 
|Python 3|PaaS 
|R|PaaS 
|SAS|IaaS 
|Vertica|IaaS (SQLDW an alternative on Azure)
|Tableau|IaaS 
|Waterline|HDInsight Edge node
|StreamSets|HDInsight Edge 
|Palantir|IaaS 
|Sailpoint|Iaas 

For more information, see the article [Hadoop components available with different HDInsight versions](../hdinsight-component-versioning.md#hadoop-components-available-with-different-hdinsight-versions)

## Customize HDInsight clusters using script actions

HDInsight provides a method of cluster configuration called a **script action**. A script action is Bash script that runs on the nodes in an HDInsight cluster and can be used to install additional components and change configuration settings.

Script actions must be stored on a URI that is accessible from the HDInsight cluster. They can be used during or after cluster creation and can also be restricted to run only on certain node types.

The script can be persisted or executed one time. The persisted scripts are used to customize new worker nodes added to the cluster through scaling operations. A persisted script might also apply changes to another node type, such as a head node, when scaling operations occur.

HDInsight provides pre-written scripts to install the following components on HDInsight clusters:

- Add an Azure Storage account
- Install Hue
- Install Presto
- Install Solr
- Install Giraph
- Pre-load Hive libraries
- Install or update Mono

> [!Note]
> HDInsight does not provide direct support for custom hadoop components or components installed using script actions.

Script actions can also be published to the Azure Marketplace as an HDInsight application.

For more information, see the following articles:

- [Install third-party Hadoop Applications on HDInsight](../hdinsight-apps-install-applications.md)
- [Customize HDInsight clusters using script actions](../hdinsight-hadoop-customize-cluster-linux.md)
- [Publish an HDInsight application in the Azure Marketplace](../hdinsight-apps-publish-applications.md)

## Customize HDInsight configs using Bootstrap

Changes to configs in the config files such as `core-site.xml`, `hive-site.xml` and `oozie-env.xml` can be made using Bootstrap. The following script is an example using Powershell:

```powershell
# hive-site.xml configuration
$hiveConfigValues = @{"hive.metastore.client.socket.timeout"="90"}

$config = New—AzureRmHDInsightClusterConfig '
    | Set—AzureRmHDInsightDefaultStorage
    —StorageAccountName "$defaultStorageAccountName.blob. core.windows.net" `
    —StorageAccountKey "defaultStorageAccountKey " `
    | Add—AzureRmHDInsightConfigValues `
        —HiveSite $hiveConfigValues

New—AzureRmHDInsightCluster `
    —ResourceGroupName $existingResourceGroupName `
    —Cluster-Name $clusterName `
    —Location $location `
    —ClusterSizeInNodes $clusterSizeInNodes `
    —ClusterType Hadoop `
    —OSType Linux `
    —Version "3.6" `
    —HttpCredential $httpCredential `
    —Config $config
```

For more information, see the article [Customize HDInsight clusters using Bootstrap](../hdinsight-hadoop-customize-cluster-bootstrap.md)

## Use edge nodes on Hadoop clusters in HDInsight to access the client tools

An empty edge node is a Linux virtual machine with the same client tools installed and configured as on the head nodes, but with no Hadoop services running. The edge node can be used for the following purposes:

- accessing the cluster
- testing client applications
- hosting client applications

Edge nodes can be created and deleted through the Azure portal and may be used during or after cluster creation. After the edge node has been created, you can connect to the edge node using SSH, and run client tools to access the Hadoop cluster in HDInsight. The edge node ssh endpoint is `<EdgeNodeName>.<ClusterName>-ssh.azurehdinsight.net:22`.

For more information, see the article [Use empty edge nodes on Hadoop clusters in HDInsight](../hdinsight-apps-use-edge-node.md)

## Use the scale-up and scale-down feature of clusters

HDInsight provides elasticity by giving you the option to scale up and scale down the number of worker nodes in your clusters. This feature allows you to shrink a cluster after hours or on weekends and expand it during peak business demands.

Cluster scaling can be automated using the following methods:

### PowerShell cmdlet

```powershell
Set-AzureRmHDInsightClusterSize -ClusterName <Cluster Name> -TargetInstanceCount <NewSize>
```

### Azure CLI

```powershell
azure hdinsight cluster resize [options] <clusterName> <Target Instance Count>
```

### Azure portal

When you add nodes to your running HDInsight cluster, any pending or running jobs won't be affected. New jobs can be safely submitted while the scaling process is running. If the scaling operations fail for any reason, the failure is gracefully handled, leaving the cluster in a functional state.

However, if you're scaling down your cluster by removing nodes, any pending or running jobs will fail when the scaling operation completes. This failure is caused by some of the services restarting during the process. To address this issue, you can wait for the jobs to complete before scaling down your cluster, manually terminate the jobs, or resubmit the jobs after the scaling operation has concluded.

If you shrink your cluster down to the minimum of one worker node, HDFS may become stuck in safe mode when worker nodes are rebooted for patching, or immediately after the scaling operation. You can execute the following command to bring HDFS out of safe mode:

```bash
hdfs dfsadmin -D 'fs.default.name=hdfs://mycluster/' -safemode leave
```

After leaving safe mode, you can manually remove the temporary files, or wait for Hive to eventually clean them up automatically.

For more information, see the article [Scale HDInsight clusters](../hdinsight-scaling-best-practices.md)

## Use HDInsight with Azure Virtual Network

Azure Virtual Networks enable Azure resources, such as Azure Virtual Machines, to securely communicate with each other, the internet, and on-premises networks, by filtering and routing network traffic.

Using Azure Virtual Network with HDInsight enables the following scenarios:

- Connecting to HDInsight directly from an on-premises network.
- Connecting HDInsight to data stores in an Azure Virtual network.
- Directly accessing Hadoop services that aren't available publicly over the internet. For example, Kafka APIs or the HBase Java API.

HDInsight can either be added to a new or existing Azure Virtual Network. If HDInsight is being added to an existing Virtual Network, the existing network security groups and user-defined routes need to be updated to allow unrestricted access to [several IP addresses](../hdinsight-extend-hadoop-virtual-network.md#hdinsight-ip-1)
in the Azure data center. Also, make sure not to block traffic to the [ports](../hdinsight-extend-hadoop-virtual-network.md#hdinsight-ports) which are being used by HDInsight services.

> [!Note]
> HDInsight does not currently support forced tunneling. Forced tunneling is a subnet setting that forces outbound Internet traffic to a device for inspection and logging. Either remove forced tunneling before installing HDInsight into a subnet or create a new subnet for HDInsight. HDInsight also does not support restricting outbound network connectivity.

For more information, see the following articles:

- [Azure virtual-networks-overview](../../virtual-network/virtual-networks-overview.md)
- [Extend Azure HDInsight using an Azure Virtual Network](../hdinsight-extend-hadoop-virtual-network.md)

## Use Azure Virtual Network service endpoints to securely connect to other Azure services

HDInsight supports [virtual network service endpoints](../../virtual-network/virtual-network-service-endpoints-overview.md) which allow you to securely connect to Azure Blob Storage, Azure Data Lake Storage Gen2, Cosmos DB, and SQL databases. By enabling a Service Endpoint for Azure HDInsight, traffic flows through a secured route from within the Azure data center. With this enhanced level of security at the networking layer, you can lock down big data storage accounts to their specified Virtual Networks (VNETs) and still use HDInsight clusters seamlessly to access and process that data.

For more information, see the following articles:

- [Virtual network service endpoints](../../virtual-network/virtual-network-service-endpoints-overview.md)
- [Enhance HDInsight security with service endpoints](https://azure.microsoft.com/blog/enhance-hdinsight-security-with-service-endpoints/.md)

## Connect HDInsight to the on-premises network

HDInsight can be connected to the on-premises network by using Azure Virtual Networks and a VPN gateway. The following steps can be used to establish connectivity:

- Use HDInsight in an Azure Virtual Network that has connectivity to the on-premises network.
- Configure DNS name resolution between the virtual network and on-premises network.
- Configure network security groups or user-defined routes (UDR) to control network traffic.

For more information, see the article [Connect HDInsight to your on-premises network](../connect-on-premises-network.md)

## Next steps

Read the next article in this series:

- [Storage best practices for on-premises to Azure HDInsight Hadoop migration](apache-hadoop-on-premises-migration-best-practices-storage.md)