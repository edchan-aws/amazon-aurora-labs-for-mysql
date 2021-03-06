# Failover to the secondary region / simulate a regional failure and DR scenario

When used in combination with in-region replicas, an Aurora cluster gives you automatic failover capabilities within the region. With Aurora Global Database, you can perform a manual failover to the cluster in your secondary region, such that your database can survive in the unlikely scenario of an entire region's infrastructure or service becoming unavailable.

Combined with an application layer that is deployed cross-region (via immutable infrastructure or [copying your AMIs cross-region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html)), you can further increase your applications' availability while maintaining data consistency.

## Creating a new table before simulating failure

>  **`Region 1 (Primary)`**

* First let us login to the Apache Superset interface again via the Primary Region instance URL.

* On the Superset navigation menu, mouse over **SQL Lab** and then click on **SQL Editor**.

* Ensure that your selected **Database** is still set to ``mysql aurora-gdb1-write`` and the **Schema** set to ``mylab``.

* Copy and paste the following SQL query and click on **Run Query**

```
DROP TABLE IF EXISTS mylab.failovertest1;

CREATE TABLE mylab.failovertest1 (
    pk INT NOT NULL AUTO_INCREMENT,
    gen_number INT NOT NULL,
    some_text VARCHAR(100),
    input_dt DATETIME,
    PRIMARY KEY (pk)
    );

INSERT INTO mylab.failovertest1 (gen_number, some_text, input_dt)
VALUES (100,"region-1-input",now());
COMMIT;

SELECT * FROM mylab.failovertest1;
```

* Note the query results.

## Failure Injection

Although we can simulate an in-region failure with the <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FaultInjectionQueries.html" target="_blank">Aurora-specific fault injection queries</a> like ```ALTER SYSTEM CRASH;``` - in this case we want to simulate a longer term and larger scale failure (however infrequent) and the best way to do this is to stop all ingress/egress data traffic in and out of the Aurora Global Database's primary DB cluster. The initial CloudFormation template created a VPC NACL with specific DENY ALL traffic that will block all ingress/egress traffic out of the associated subnets. This will essentially emulate a wider failure that will render our primary region DB cluster unavailable.

>  **`Region 1 (Primary)`**

* Open <a href="https://console.aws.amazon.com/vpc" target="_blank">VPC</a> in the AWS Management Console. Ensure you are in your assigned region.

* Within the VPC console, scroll down on the left menu and select **Network ACLs**. This will bring you to the list of NACLs that are in your VPC. You should see **gdb1-nacl-denyall** , and that it is not currently associated with any subnets.

    * Click on the **gdb1-nacl-denyall** NACL, and review both the Inbound Rules and Outbound Rules. You should see that they're set to *DENY* for ALL traffic.

* Click on **Actions** menu, and then select **Edit subnet associations**
    <span class="image">![NACLs Review](failover-nacl1.png)</span>

* As the Aurora DB cluster is set to use the private subnets (governed by RDS DB subnet group), all of which labeled with the prefix **gdb1-prv-sub-X**, select all subnets that begin with that prefix description. You can also simply use the search box to filter on any subnets with the name **prv** and then select them. Next, click on the **Edit** button to confirm the associations. (Note: you may have to drag the first column wider to fully see the subnet names)
    <span class="image">![NACLs Review](failover-nacl2.png)</span>

* Once associated, the NACL rules take effect immediately and will render the resources within the private subnets unreachable. 

* Go back to your browser tab with your primary region's Apache Superset SQL Editor. Click on the &#8644; refresh button next to **Schema**.

* You will notice that you can no longer access the primary cluster, and the schema refresh will eventually time out. We have successfully injected failure to render our Primary DB Cluster unreachable.

## Promote Secondary DB Cluster

As we are simulating a prolonged regional infrastructure or service level failure, in which point we will opt to perform a regional failover for our application and database stacks. We will take the next steps on promoting the secondary DB cluster to a regional master DB cluster.

>  **`Region 2 (Secondary)`**

* Open <a href="https://console.aws.amazon.com/rds" target="_blank">RDS</a> in the AWS Management Console. Ensure you are in your assigned region.

* Within the RDS console, select **Databases** on the left menu. This will bring you to the list of Databases already deployed. You should see **gdb2-cluster**  and **gdb2-node1** .

    ??? tip "Why does the status of my primary DB cluster and DB instance still report as <i>Available</i>?"
        You might also notice that in your RDS console it will still report the primary region DB cluster and DB instance still as *Available*, that is because we are merely simulating a failure by blocking all networking access via NACLs, and the blast radius of such a simulation is limited only to your AWS Account, specifically to your VPC and its affected subnets; the RDS/Aurora service and its internal health checks are provided by the service control plane itself and will still report the DB cluster and DB instance as healthy because there is no *real outage*.

* Select the secondary DB Cluster **gdb2-cluster**. Click on the **Actions** menu, then select **Remove from Global**
    <span class="image">![Aurora Promote Secondary](failover-aurora-promote1.png)</span>
    * A message will pop up asking you to confirm that this will break replication from the primary DB cluster. Confirm by clicking on **Remove and promote**.

* The promote process should take less than 1 minute. Once complete, you should be able to see the previously secondary DB cluster is now labeled as **Regional** and the DB instance is now a **Writer** node.
    <span class="image">![Aurora Promote Secondary](failover-aurora-promote2.png)</span>

* Click on the newly promoted DB cluster. Under the **Connectivity and security** tab, the *Writer* endpoint should now be listed as *Available*. Copy and paste the endpoint string into your notepad as we prepare for failover on the application stack and adding this endpoint as the new writer.  
    <span class="image">![Aurora Promote Secondary](failover-aurora-promote3.png)</span>

## Restore Application Write Access

>  **`Region 2 (Secondary)`**

* Log in to the Secondary Region instance of Apache Superset. Use the **Apache Superset Secondary URL** from your notes in the previous step.

* In the Apache Superset navigation menu, mouse over **SQL Lab**, then click on **SQL Editor**.

* Ensure that your selected **Database** is still set to ``mysql aurora-gdb2-read`` and the **Schema** set to ``mylab``.

* Copy and paste the following SQL query and click on **Run Query**

```
SELECT * FROM mylab.failovertest1;

```

* Note the query results, this should return the new table and record we entered into the Database shortly before the simulated failure.

* Let's try to insert a new record to this table. Copy and paste the following DML query and click on **Run Query** - what do you expect the results to be?
    
```
INSERT INTO mylab.failovertest1 (gen_number, some_text, input_dt)
VALUES (200,"region-2-input",now());
COMMIT;

SELECT * FROM mylab.failovertest1;    
```

* Remember our secondary region Apache Superset instance previously utilized the specific Reader endpoint for the Global Database cluster, and the Writer endpoint was previously unavailable as a Secondary DB Cluster. As this database backend is now promoted to a writer, we can add a new datasource using the new writer endpoint to allow this instance of Apache Superset to serve write requests.

   1. In the Apache Superset navigation menu, mouse over **Sources**, then click on **Databases**.
      <span class="image">![Superset Source Databases](../biapp/superset-source-db.png)</span>

   1. Near the top right, click on the green  **+** icon to add a new database source.

   1. Change the below values and press **Save** when done:

      Field | Value and Description
      ----- | -----
      Database | <pre>aurora-gdb2-write</pre> <br> This will be the friendly name of our Aurora Database in Superset<br>&nbsp;
      SQLAlchemy URI | <pre>mysql://masteruser:<b>auroragdb321</b>@<b><i>[Replace with Secondary Writer Endpoint]</i></b>/mysql</pre> <br> Replace the endpoint with the Secondary Writer Endpoint we have gathered previously. The password to connect to the database should remain as ```auroragdb321``` unless you have changed this value during CloudFormation deployment. Click on **Test Connection** to confirm.<br>&nbsp;
      Expose in SQL Lab | &#9745; (Checked)
      Allow CREATE TABLE AS | &#9745; (Checked)
      Allow DML | &#9745; (Checked)

      ![Superset GDB2 Write Settings](../biapp/superset-gdb2w.png)

* Return to Apache Superset SQL Editor. Ensure that your selected **Database** is set to the new ``mysql aurora-gdb2-write`` and the **Schema** set to ``mylab``.

* Copy and paste the following DML query again and click on **Run Query**

```
INSERT INTO mylab.failovertest1 (gen_number, some_text, input_dt)
VALUES (200,"region-2-input",now());
COMMIT;

SELECT * FROM mylab.failovertest1;    
```

* You should notice both your previous results before failover and the new record. Your application is now serving both read and write queries and serve your users as normal, during a region-wide disruption!

## Checkpoint

You have just performed a failover operation of your application from its primary region to secondary, during a simulated regional infrastructure or service level disruption. This allows you to build reliable applications serving your customers anywhere in the world that can be resilient and maintain its uptime in the face of a catastrophic disaster.

![Failover Diagram](failover-arch2.png)

* In a real world scenario, you might want to front load your application tier with an application load balancer. Should you want to have seamless transition for your endpoint that handles write DML queries, you can also combine your applications with Route53 Active-Passive failover. These configurations are outside the scope of this particular workshop, but you can find more information on such architecture on AWS website:
    * DNS Failover Types with Route53: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html#dns-failover-types-active-passive
    * Creating Route53 DNS Health Check: https://aws.amazon.com/premiumsupport/knowledge-center/route-53-dns-health-checks/
    * AWS This is my Architecture series - Multi-Region High-Availability Architecture: https://www.youtube.com/watch?v=vGywoYc_sA8

If you are up for another challenge, go to the optional step of [Failback](../failback/index.md).

