#Tag Based Security with Atlas + Ranger
##Install Pre-requisites

###Install HBase
Atlas uses HBase for its graph database store. If HBase is not already installed, HBase will need to be added to the cluster.
- In Ambari, use the Add Servcie wizard to add HBase to the cluster.
- Choose the HBase Master node and assign Region Servers to the appropriate worker nodes in the cluster.
- Enable the HBase-Ranger plugin: Ambari -> Ranger -> Configs -> Ranger Plugin -> HBase Ranger Plugin -> On
- Restart all neccessary components in Ambari

###Install Kafka
Atlas uses Kafka to queue messages for tracking lineage and audit usage as well as for transferring infmriaton about assets within the environment.
- In Ambari, use the Add Servcie wizard to add Kafka to the cluster.
- Enable the Kafka-Ranger plugin: Ambari -> Ranger -> Configs -> Ranger Plugin -> Kafka Ranger Plugin -> On
- Restart all neccessary components in Ambari

##Install Atlas
Once Kafka and HBase are installed and the Ranger plugins are enabled, the Atlas service can be installed and configured.
- In Ambari, use the Add Servcie wizard to add Atlas to the cluster.
- At the configuration step, set the authentication parameters for Atlas. Use the following parameter values:
  - Authentication Type: `AD`
  - atlas.authentication.method.ldap.ad.url: `ldap://ad01.lab.hortonworks.net:389`
  - Domain Name: `LAB.HORTONWORKS.NET`
  - atlas.authentication.method.ldap.ad.base.dn: `dc=lab,dc=hortonworks,dc=net`
  - atlas.authentication.method.ldap.ad.bind.dn: `cn=ldap-reader,ou=ServiceUsers,dc=lab,dc=hortonworks,dc=net`
  - atlas.authentication.method.ldap.ad.bind.password: `BadPass#1`

![Image](images/atlas-auth-config.png?raw=true)

- In the Advanced ranger-atlas-plugin-properties configuration page, check the box for Enable Ranger for Atlas

![Image](images/enable-ranger-atlas.png?raw=true)

- Restart all affected services
- Troubleshooting
  - Not enabling the Ranger Atlas Plugin may cause the Atlas Metadata Server to not start properly. If this happens, check the logs for the following error:

    ```
    ERROR Java::OrgApacheHadoopHbaseIpc::RemoteWithExtrasException: org.apache.hadoop.hbase.coprocessor.CoprocessorException: HTTP 400 Error: atlas is Not Found
    ```
    
##Configure Ranger Policies
Ranger policies must be configured to ensure that the hadoopadmin user can administer Atlas. Policies are also created when Atlas is installed for verious components (HBase, Kafka). The creation of these policies is not 100% automated, so these policies need to be verified before continuing.

- Verify the HBase policies in Rager.
  - Policy for atlas_titan table:
    - User: `atlas`, Privileges:  `Read, Create, Admin, Write`

  ![Image](images/atlas-titan-policy.png?raw=true)
  
  - Policy for ATLAS_ENTITY_AUDIT_EVENTS table:
    - User: `atlas`, Privileges:  `Read, Create, Admin, Write`
    
  ![Image](images/atlas-audits-policy.png?raw=true)

- Verify the Kafka policies in Ranger.
  - Policy for ATLAS_HOOK topic:
    - User: `atlas`, Privileges: `Consume, Create`
    - Group: `public`, Privileges: `Publish, Create`

    ![Image](images/atlas-hook-policy.png?raw=true)
    
  - Policy for ATLAS_ENTITIES topic:
    - User: `atlas`, Privileges: `Publish, Create`
    - Group: `public`, Privileges: `Consume, Create`

    ![Image](images/atlas-entities-policy.png?raw=true)

- Add the `hadoopadmin` user to all of the default Atlas policies in Ranger

![Image](images/ranger-atlas-policies.png?raw=true)

##Import Hive Objects into Atlas
Atlas does not import metadata for Hive tables at install time. If there are any existing Hive tables, then a script needs to be run to import those objects into the Atlas Metadata Server.

On the node where the Atlas Metadata Server is running, execute the following:
```
sudo -u atlas
kinit -kt /etc/security/keytabs/atlas.service.keytab atlas/$(hostname -f)@LAB.HORTONWORKS.NET
/usr/hdp/current/atlas-server/hook-bin/import-hive.sh
```

##Correct the ScemaLayoutView.js File
The Atlas Metadata Server in HDP 2.5.0 ships with an incorrect version of the ScemaLayoutView.js file in the Atlas webapp structure. Replace this file and clear the browser cache to get a proper representation in the Schema tab of a table's description in Atlas.
- Download the [ScemaLayoutView.js](ScemaLayoutView.js) file to the node running the Atlas Metadata Server.
- Backup the original file and replace it with the new file

```
cd /usr/hdp/current/atlas-server/server/webapp/atlas/js/views/schema/
mv SchemaLayoutView.js SchemaLayoutView.js.old
mv ~/SchemaLayoutView.js ./SchemaLayoutView.js
chown atlas:hadoop SchemaLayoutView.js
chmod 644 SchemaLayoutView.js
```

##View Atlas Metadata and Create PII Tag
- Login to the Atlas UI (Ambari -> Atlas -> Quick Links -> Atlas UI) as the `hadoopadmin` user
- Find Hive tables
  - Click on Search
  - Move the toggle to DSL
  - In the `Search For` box, type `hive_table` and click the Search button
  
  ![Image](images/atlas-table-search.png?raw=true)
  
  - Click on the table names and explore the Atlas UI. If the SchemaLayoutView.js was replaced properly, the `Schema` tab should show the column names for the table.

- Crate a PII tag in Atlas
  - Click on Tags
  - Click on Create Tag
  - Give the tag the name `PII`
  - Give the tag the description `Private Identifiable Data`
  - Click `Create`
  - The new PII tag will appear below the search box
  
  ![Image](images/atlas-pii-tag.png?raw=true)

- Assign the PII tag to the salary column for sample_07
  - Go back to the Atlas Search 
  - Select the sample_07 table
  - Browse to the Schema tab
  
  ![Image](images/atlas-sample07-schema.png?raw=true)
  
  - Click the `+` sign in the `Tags` column next to the `salary` field
  - Select the `PII` tag from the drop down and click `Add`
  
  ![Image](images/atlas-sample07-pii.png?raw=true)

The salary column of the sample_07 table is now tagged as `PII` data in Atlas.

##Create Ranger Tag Policies
Now that the tag has been created and assigned to an asset in Atlas, a policy can be created in Ranger to limit access to this PII data.
###Setup Ranger to use Tag Based Policies
- Login to Ranger as an administrative user (`admin/admin`)
- Navigate to Access Manager -> Tag Based Policies

![Image](images/ranger-tag-policy.png?raw=true)

- Click the `+` next to `Tag` to create a new tag service
 
![Image](images/ranger-create-tag-service.png?raw-true)

- Create a new tag service with a name <cluster_name>_tags (e.g. if the cluster name is sme-security-11, then use sme-security-11_tags as the name)
- Click `Add`

###Create a Tag Based Policy for the PII Tag
- In Ranger, navigate to Access Manager -> Tag Based Policies
- Select the <cluster_name>_tags service to create a tag based policy
- Click `Add New Policy` 
- Create a policy with the following attributes:
  - Policy Name: `PIIPolicy`
  - TAG: `PII`
  - Allow conditions:
    - Group: `hr`, Allowed Operations: `select, update` for `hive` component

    ![Image](images/ranger-pii-allow-perms.png?raw=true)

 - Deny conditions:
    - Group: `sales`, Denied Operations: all for `hive` component
    
    ![Image](images/ranger-pii-deny-perms.png?raw=true)

  - Overall, the policy should look like:
  
  ![Image](images/ranger-pii-tag-policy.png?raw=true)

###Associate Tag Policy Service with Hive Policy Service
Now that the tag based policy service has been created, the Hive policy service needs to be configured to use the tag policies.
- Navigate to Access Manager -> Resource Based Policies 
- Edit the Hive policy service by clicking on the gray `Edit` button next to the service name

![Image](images/edit-hive-service.png?raw=true)

- In the `Select Tag Service` drop down, select the name of the tag policy service created above

![Image](images/hive-service-add-tags.png?raw=true)

- Click `Save`

###Setup Table Access Policy
Create a Hive policy in Ranger that gives both the sales and hr groups access to look at and update the sample_07 table.
- In Ranger, navigate to Access Manager -> Resource Based Policies
- Click on the Hive policy service
- Click `Add New Policy` 
- Add a policy that gives access to the sample_07 table:
  - Poicy Name: `Sample07Policy`
  - Database: `default`
  - Table: `sample_07`
  - Hive Column: `*`
  - Group: `hr`, Permissions: `select, update`
  - Group: 'sales', Permissions: `select, update`
  ![Image](images/hive-sample07-policy.png?raw=true)

###Give User Groups Access to the Hive View
- In Ambari, navigate to User Name Dropdown -> Manage Ambari -> Views -> Hive View
- Grant permissions to the `sales` and `hr` groups to use the Hive View

![Image](images/hive-view-perms.png?raw=true)

##Test access to data tagged as PII
Now that all of the policies and associations are complete, access to the tagged data componets can be tested. This can be completed either using beeline, or using the Hive View (if properly configured).

###Show Access for HR Users
The HR users should be able to see all columns in the sample_07 table since they were granted Allow permissions to the tag based policy.
- Login to Ambari as `hr1/BadPass#1`
- Click on `Hive View`
- Run a query to view the data in the sample_07 table:
```
select * from sample_07 limit 50;
```

![Image](images/hive-hr1-query-succeed.png?raw=true)

###Show Access for Sales Users
The Sales users should be able to see all columns of the sample_07 table except the salary column.
- Login to Ambari as `sales1/BadPass#1`
- Click on `Hive View`
- Run a query to view all data in the sample_07 table:

```
select * from sample_07 limit 50;
```

![Image](images/hive-sales1-query-fail.png?raw=true)

- Notice that this query fails with the following error:

```
java.lang.Exception: org.apache.hive.service.cli.HiveSQLException: Error while compiling statement: FAILED: HiveAccessControlException Permission denied: user [sales1] does not have [SELECT] privilege on [default/sample_07/code,description,salary,total_emp]
```

- Run another query against the sample_07 table excluding the salary column:
```
select code, description, total_emp from sample_07 limit 50;
```

![Image](images/hive-sales1-query-succeed.png?raw=true)

###View Ranger Audits for Tag Policy
Ranger logs access successes and failures in the Audit trail.
- In Ranger, navigate to Audit -> Access
- Add a filter to see access failures for the sales1 user:
  - Service Type: `HIVE`
  - Result: `Denied`
  
  ![Image](images/ranger-sales1-denied.png?raw=true)

- Remove the filter on Result to view successes

  ![Image](images/ranger-sales1-allowed.png?raw=true)


##Optional: Enable Taxonomy Features in Atlas
The taxonomy features in Atlas are Technical Preview as of HDP 2.5.0. To enable these features, add a paremeter in Ambari
Add atlas.feature.taxonomy.enable=true
