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

###Install Atlas





Enable Ranger plugin (Metadata server won't start till this is done)
give atlas user access to kafka topics

| Topic          | User/Group    | Permissions      |
| :------------- |:-------------:| :----------------|
| ATLAS_HOOK     | atlas         | consume, create  |
|                | public        | publish, create  |
| ATLAS_ENTITIES | atlas         | consume, create  |
|                | public        | publish, create  |


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
