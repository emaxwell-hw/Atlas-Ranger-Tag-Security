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





Grant privileges to the admin user on the default Atlas policies.
Replace the SchemaLayoutView.js file (/usr/hdp/current/atlas-server/server/webapp/atlas/js/views/schema/SchemaLayoutView.js)
- clear browser history (or restart browser) and reload page

##Step 2: Setup Authentication for Atlas
AD configs

##Step 3: Import Hive Table Information
su - atlas
kinit -kt /etc/security/keytabs/atlas.service.keytab atlas/ip-172-30-0-26.us-west-2.compute.internal@SEC11.HORTONWORKS.COM
/usr/hdp/current/atlas-server/hook-bin/import-hive.sh

##Step 4: Enable Taxonomy Features
Custom application-properties
Add atlas.feature.taxonomy.enable=true

##Step 5: Create Tag Policy Service
Ranger -> Tag Based Policies -> + -> <cluster>_tags

##Step 6: Associate tag policy service with Hive policy service
Service Manager -> Edit Hive Servcie -> Select Tag Service -> choose tag service
