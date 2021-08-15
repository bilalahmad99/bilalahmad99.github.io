---
layout: default
---

## Our disaster recovery approach on Google Cloud SQL

Firstly we don't have any on-premise database instances so it makes our life much easier. Few things to note before designing your disaster recovery approach is RTO (Recovery time objective) and RPO (Recovery point objective) metrics. Both of these can vary from service to service as well as business to business. RTO is the maximum acceptable length of time that your application can be offline, and RPO is s the maximum acceptable length of time during which data might be lost from your application due to a major incident. The values of these you set yourself as part of your SLAs (Service level agreements). And over time you improve them and make them less as much as possible.
Here is our approach:
#### All production cloud sql instances must have HA enabled

Its quite easy to configure HA on both gcloud console and through automation (we use terraform btw).
gcloud instructions [here](https://cloud.google.com/sql/docs/mysql/configure-ha)
terraform [here](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_database_instance#availability_type)

One tip is to try and enable HA while creating new instance when it does not have data. Enabling it later on existing SQL causes a down time. And the down time depends on how much data is there in the SQL at the point you are enabling it.

#### Enable automated backups and point-in-time recovery for all instances

Gcloud provides option to setup automated backups at your preferred time. You can enable it easily and its always good to keep the time slot as the one during which database has less activity because while creating backup the performance of database take a hit.
more info [here](https://cloud.google.com/sql/docs/postgres/backup-recovery/backing-up)

Gcloud also offers point-in-time recovery of SQL. This is by default enabled. more info [here](https://cloud.google.com/sql/docs/postgres/backup-recovery/pitr#pitr)

Apart from automated backups, its always good to create many on-demand backups of your crucial SQL instances. Best way is to automate the process. We have it as a cronjob running in our Kubernetes. 

Tip: store your automated on-demand backups at a different place then your regular automated backups.

#### Every instance should have same-region and cross-region replica.

Same region replica is for all the apps that does read operations to database and cross region replicas are for disaster recovery case. in case there is an issue with gcloud region. Choosing the right size of replicas is also crucial, it does not have to be the same as the master instance but it needs to big enough to handle load once it is promoted (in case of incident)
more info [here](https://cloud.google.com/blog/products/databases/introducing-cross-region-replica-for-cloud-sql)

#### Simulate a disaster and recovery

Nothing will give you more confidence in your disaster recovery strategy then actually having an incident and recovering from it. The next best is simulating it. :D

Tip: Do a full process of incident protocol while simulating e.g. going for the backup or cross region replica and making it primary and then making apps use it. One thing is when you promote a replica you have to setup HA and automated backups etc for it manually e.g. its not done by default. So do it right after creating replica rather then doing at later stage and then taking another hit of downtime.


[back](../)