# Azure Databricks Best practices

## [Workspace Administration](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-best-practices/2-workspace-admin)

1. **Separating Development, Staging and Production Azure Databricks environments**.

    - Deployment throug Azure portal, Azure Resource Manager templates or [Terraform](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/databricks_workspace).
    - One deployment produces exactly one Workspace. (file browser, notebooks, tables, clusters, DBFS storage, etc)
    - Uses AAD as the exclusive Identity Provider, but ADB comes with its own user management interface (create users and groups and assign priviledges) combined with [SCIM](https://docs.microsoft.com/en-gb/azure/databricks/administration-guide/users-groups/scim/aad) you can sync AAD to ADB.
    - Multiple clusters can exist within a workspace, and there's a one-to-many mapping between a Subscription to Workspaces, and further, from one Workspace to multiple Clusters.

1. **Map workspaces to business divisions**

1. **Deploy workspaces in multiple subscriptions to honor Azure capacity limits**

    - Databricks workspace limits:
        - The maximum number of jobs that a workspace can create in an hour is 1000
        - At any time, you cannot have more than 150 jobs simultaneously running in a workspace
        - There can be a maximum of 150 notebooks or execution contexts attached to a cluster
        - There can be a maximum of 1500 Azure Databricks API calls/hour.
    - Azure subscription limits:
        - Storage accounts per region per subscription: 250
        - Maximum egress for general-purpose v2 and Blob storage accounts (all regions): 50 Gbps
        - Virtual Machines (VMs) per subscription per region: 25,000
        - Resource groups per subscription: 980

1. **High availability / Disaster recover (HA/DR)**

    Within each subscritption, consider the following best practices for HA/DR
    - Deploy Azure Databricks in two paired Azure regions, ideally mapped to different control plane regions (East US2 and West US2)
    - Use Azure Traffic Manager to load balance and distribute API request between two deployments, when the platform is primarily being used in a backend non-interactive mode

1. **Additional considerations**

    - Define workspace level tags with propagate to initially provisioned resources in managed resource group.

## [List security best practices](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-best-practices/3-security)

1. **Consider isolating each workspace in its own VNet**

    - Deploy each Workspace in its own spoke VNet.
    - Put all the common networking resources in a central hub VNet, such as your custom DNS server.
    - Join the Workspace spokes with the central networking hub using VNet Peering.

1. **Do not store any production data in Default Databricks Filesystem (DBFS) Folders**

    - The lifecycle of default DBFS is tied to the Workspace. Deleting the workspace will also delete the default DBFS and permanently remove its contents.
    - One can't restrict access to this default folder and its contents.

1. **Always hide secrets in a key vault**

    If using Azure Key Vault, create separate AKV-backed secret scopes and corresponding AKVs to store credentials pertaining to different data stores. This will help prevent users from accessing credentials that they might not have access to. Since access controls are applicable to the entire secret scope, users with access to the scope will see all secrets for the AKV associated with that scope.

    - [Create an Azure Key Vault-backed secret scope](https://docs.microsoft.com/en-gb/azure/databricks/security/access-control/secret-acl)
    - [Best practices for creating secret scopes](https://docs.microsoft.com/en-gb/azure/databricks/security/secrets/secret-scopes)
    - [Example of using a secret in a notebook](https://docs.microsoft.com/en-gb/azure/databricks/security/secrets/example-secret-workflow)

1. **Access control - Azure Data Lake Storage (ADLS) passtrhough**

    When enabled, authentication automatically takes place in Azure Data Lake Storage (ADLS) from Azure Databricks clusters using the same Azure Active Directory (Azure AD) identity that one uses to log into Azure Databricks. Commands running on a configured cluster will be able to read and write data in ADLS without needing to configure service principal credentials. Any ACLs applied at the folder or file level in ADLS are enforced based on the user's identity. [How to here](https://docs.microsoft.com/en-us/azure/databricks/security/credential-passthrough/adls-passthrough#single-user)

1. **Configure audit logs and resource utilization metrics to monitor activity**

    An important facet of monitoring is understanding the resource utilization in Azure Databricks clusters. In order to get utilization metrics of an Azure Databricks cluster, you can stream the VM's metrics to an Azure Log Analytics Workspace (see Appendix A) by installing the Log Analytics Agent on each cluster node.

1. **Querying VM metrics in Log Analytics once you have started the collection using the above document**

1. **Additional considerations**
    - Configure encryption-at-rest for Blob Storage and ADLS, preferably by using customer-managed keys in Azure Key Vault.

## [Describe tools and integration best practices](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-best-practices/4-tools-integration)

1. **Favor cluster scoped init scripts over global and named scripts**.
    - [Here is how to init scripts](https://docs.microsoft.com/en-gb/azure/databricks/clusters/init-scripts)
    - Create cluster:
        - [CLI](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/cli/clusters-cli)
        - [API](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/api/latest/clusters)
        - [Terraform](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/terraform/)
            - [TF provider](https://registry.terraform.io/providers/databrickslabs/databricks/latest/docs/guides/workspace-management) (No SLA by Microsoft, [here is the project](https://github.com/databrickslabs/terraform-provider-databricks/issues/new/choose))

1. **Use cluster log delivery feature to manage logs**
    - [Cluster Log Delivery](https://docs.microsoft.com/en-gb/azure/databricks/clusters/configure#cluster-log-delivery) to a blob store as opposed to dbfs.

1. **Additional considerations**
    - Use [Azure Data Factory](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/data-pipelines#azure-data-factory) to orchestrate pipelines/workflows.
      - [Transform data](https://docs.microsoft.com/en-us/azure/data-factory/transform-data-databricks-notebook) by running a Databricks notebook
    - Connect your IDE or custom applications to Azure Databricks clusters using [Databricks-Connect](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/databricks-connect)
    - Sync notebooks with [Azure DevOps](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/ci-cd/ci-cd-azure-devops) for seamless version control.
    - Use Databricks [CLI](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/cli/clusters-cli) for CI/CD from relevant enterprise tools/products or to integrate with other systems like on prem SCM or Library Repos, etc.
    - Use [Library Utilities](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/databricks-utils#library-utilities) to install pyton libraries scopred at notebook level.

## [Databricks Runtime](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-best-practices/5-databricks-runtime)

1. **Tune shuffle for optimal performance**
    - Narrow transformation
        - Input and output stays in same partition
        - No data movement is needed
    - Wide transformation
        - Input from other partitions are required
        - Data shuflling is needed before processing.
    - You've got two control knobs of a shuffle you can use to optimize:
        - The number of partitions being shuffled:

        ```spark
        spark.conf.set("spark.sql.shuffle.partitions", 10)
        ```

        - The number of partitions that you can compute in parallel.
            - This is equal to the number of cores in a cluster.

        These two determine the partition size, which we recommend should be in the Megabytes to 1-Gigabyte range. If your shuffle partitions are too small, you may be unnecessarily adding more tasks to the stage. But if they are too big, you may get bottlenecked by the network

1. **Partition your data**
    - Evenly distributed data across all partitions (date is the most common)
    - 10 s of GB per partition (~10 to ~50 GB)
    - Small data sets should not be partitioned
    - Beware of over partitioning

## [Understand cluster best practices](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-best-practices/6-cluster)

We need to understand how different types of jobs demand different types of cluster resources.

- **Machine Learning** - To train machine learning models its required cache all of the data in memory. Consider using memory optimized VMs. Also use storage optimized instances for larg data sets. To size the cluster, take a % of the data set -> ache it -> see how mush memory it used -> extrapolate that to the rest of the data.

- **Streaming** - Make sure that the processing rate is just above the input rate at peak times of the day. Depending on peak input rate times, consider computing optimized VMs.

- **Extract, Transform and Load (ETL)** - Data sice and deciding how fast a job needs to be will be a leading indicator. Spark doesn't always require data to be loaded into memory in order to execute transformation, but you'll at least need to see how large the task sizes are on shuffles anc compare that to the task throughput you'd like. To analyze the performance of these jobs start with basics and check if the job is by CPU, network, or local I/O, and go from there. Consider using a general purpose.

- **Interactive / Development Workloads** - The ability for a cluster to auto scale is most important for these type of jobs. Take advantage of [autoscaling feature](https://docs.microsoft.com/en-gb/azure/databricks/clusters/configure#autoscaling).

When it comes to taxonomy, ADB clusters are divided along the notions of "type" and "mode". Two types; _Interactive_, created using UI and  Clusters API, and _Job Clusters_ created using Jobs API. Further, each cluster can be of two modes: Standard and high Concurrency. In addition to Autoscaling features, use [auto-termination](https://docs.microsoft.com/en-gb/azure/databricks/clusters/clusters-manage#cluster-terminate) when applicable.

### Choosing the VM type

### Cluster sizing starting points

The broad approach you should follow for sizing is:

1. Develop on a medium-sized cluster of 2-8 nodes, with VMs matched to workload class as explained earlier.
1. After meeting functional requirements, run end to end test on larger representative data while measuring CPU, memory and I/O used by the cluster at an aggregate level.
1. Optimize cluster to remove bottlenecks found in step 2

    - CPU bound: add more cores by adding more nodes
    - Network bound: use fewer, bigger SSD backed machines to reduce network size and improve remote read performance
    - Disk I/O bound: if jobs are spilling to disk, use VMs with more memory.

Repeat steps 2 and 3 by adding nodes and/or evaluating different VMs until all obvious bottlenecks have been addressed.

Performing these steps will help you to arrive at a baseline cluster size which can meet SLA on a subset of data. In theory, Spark jobs, like jobs on other Data Intensive frameworks (Hadoop) exhibit linear scaling. For example, if it takes 5 nodes to meet SLA on a 100 TB dataset, and the production data is around 1PB, then prod cluster is likely going to be around 50 nodes in size. You can use this back of the envelope calculation as a first guess to do capacity planning. However, there are scenarios where Spark jobs don't scale linearly. In some cases this is due to large amounts of shuffle adding an exponential synchronization cost (explained next), but there could be other reasons as well. Hence, to refine the first estimate and arrive at a more accurate node count we recommend repeating this process 3-4 times on increasingly larger data set sizes, say 5%, 10%, 15%, 30%, etc. The overall accuracy of the process depends on how closely the test data matches the live workload both in type and size.

### Rules of thumb

- Fewer big instances > more small instances
  - Reduce network shuffle; Databrikcs has 1 executor / machine
  - Applies to batch ETL mainly
  - Not set in stone, and reverse would make sense in many cases
- Size based on the number of tasks initially, tweak later
  - Run the job with a small cluster to get idea of # of tasks (use 2-3x tasks per core for base sizing)
- Choose based on workload (Probably start with F-series or DSv2)
  - ETL with full file scans and no data reuse - F / DSv2
  - ML workload with data caching - DSv2 / F
  - Data Analysis - L
  - Streaming - F

### Workload requires caching (like machine learning)

- Look at the Storage tab in Spark UI to see if the entirety of the training dataset is cached
  - Fully cached with room to spare -> fewer instances
  - Partially cached
        - Almost cached? -> Increase the cluster size
        - Not even close to cached -> Consider L series or DSv2 memor-optimized
  - Check to see if persist is MEMORY_ONLY, or MEMORY_AND_DISK
  - Spill to disk with SSD isn't so bad

### ETL and analytics workloads

- Are we compute-bound?
  - Check CPU Usage
  - Only way to make faster is more cores
- Are we network-bound?
  - Check for high spikes before compute haevy steps
  - Use bigger/fewer machines to reduce the shuffle
  - Use an SSD backed instance for faster remote reads
- Are we spilling a ton?
  - Check Spark SQL tab for spill
- Use L-series
- Or use more memory
