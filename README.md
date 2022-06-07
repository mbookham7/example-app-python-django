# Test the Multi Region CockroachDB with Django Application

In this repo I have taken the Django example from the CockroachDB docs [here](https://www.cockroachlabs.com/docs/stable/build-a-python-app-with-cockroachdb-django.html) and adapted it to take advantage of the CockroachDB multi region capabilities.

I have updated the code to allow for an additional field to be submitted to the customers api. This additional field allows you to specify the cloud that the request is being posted from. This allows us to then pin the data to a specific locality in our CockroachDB cluster.

This is a very simple example purely to demonstrate the CockroachDB multi regional capabilities.

Below are step by step instructions to deploy the application across three Kubernetes clusters.


1. Create the required environment variables.
```
export azregion="uksouth"
export clus1="mb-aks-cluster-1"
export aws_region="eu-west-1"
export clus2="arn:aws:eks:eu-west-1:541263489771:cluster/mb-eks-cluster-1"
export gcp_region="europe-west4"
export clus3="gke_cockroach-bookham_europe-west4_mb-gke-cluster-1"
```

2. Create a folder called certs and add your `ca.crt`,`client.root.crt` and `client.root.key` to this folder. we will need these to connect to the database server.


3. Deploy a pod running the CockroachDB CLI so that the Database for application can be created.
```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $azregion --context $clus1
```

4. Exec into the pod to access the SQL prompt.
```
kubectl exec -it cockroachdb-client-secure \
-n $azregion \
--context $clus1 \
-- ./cockroach sql \
--certs-dir=/cockroach-certs \
--host=cockroachdb-public
```

5. Create the database called django.
```
CREATE DATABASE django;
\q
```

6. Deploy the application into each of the regions.
```
kubectl apply -f ./kubernetes/deployment.yaml --context $clus1 --namespace $azregion
kubectl apply -f ./kubernetes/deployment.yaml --context $clus2 --namespace $aws_region
kubectl apply -f ./kubernetes/deployment.yaml --context $clus3 --namespace $gcp_region
```

7. Retrieve the Loadbalancer IP address for each of the regions.
```

az_app_ip=$(kubectl get svc django-service --context $clus1 --namespace $azregion -o json | jq -r '.status.loadBalancer.ingress[0].ip')
aws_app_ip=$(kubectl get svc django-service --context $clus2 --namespace $aws_region -o json | jq -r '.status.loadBalancer.ingress[0].hostname')
gcp_app_ip=$(kubectl get svc django-service  --context $clus3 --namespace $gcp_region -o json | jq -r '.status.loadBalancer.ingress[0].ip')
```

8. Use the API of the application to add three entries into the Database. You will notice the second field is cloud with a different value to indicate the cloud it was deployed into.
```
curl --header "Content-Type: application/json" \
--request POST \
--data '{"name":"Carl", "cloud":"azure"}' http://$az_app_ip:8000/customer/

curl --header "Content-Type: application/json" \
--request POST \
--data '{"name":"Mike", "cloud":"aws"}' http://$aws_app_ip:8000/customer/

curl --header "Content-Type: application/json" \
--request POST \
--data '{"name":"Dan", "cloud":"gcp"}' http://$gcp_app_ip:8000/customer/
```

9. Retrieve the data from from the database to ensure that it has been written.
```
curl http://$az_app_ip:8000/customer/
```

10. Set the primary region for the database django.
```
ALTER DATABASE django PRIMARY REGION "uksouth";
ALTER DATABASE django ADD REGION "eu-west-1";
ALTER DATABASE django ADD REGION "europe-west4";
```

11. For the table `cockroach_example_customers`, the right table locality for optimizing access to their data is `REGIONAL BY ROW`. These statements use a `CASE` statement to put data for a given cloud in the right region.
```
ALTER TABLE cockroach_example_customers ADD COLUMN region crdb_internal_region AS (
  CASE WHEN cloud = 'aws' THEN 'eu-west-1'
       WHEN cloud = 'azure' THEN 'uksouth'
       WHEN cloud = 'gcp' THEN 'europe-west4'
  END
) STORED;
ALTER TABLE cockroach_example_customers ALTER COLUMN REGION SET NOT NULL;
ALTER TABLE cockroach_example_customers  SET LOCALITY REGIONAL BY ROW AS "region";
```

12. Next, run a replication report to see which ranges are still not in compliance with your desired domiciling.
```
SELECT * FROM system.replication_constraint_stats WHERE violating_ranges > 0;
```

13. Next, run the query suggested in the Replication Reports documentation that should show which database and table names contain the violating_ranges.
```
WITH
    partition_violations
        AS (
            SELECT
                *
            FROM
                system.replication_constraint_stats
            WHERE
                violating_ranges > 0
        ),
    report
        AS (
            SELECT
                crdb_internal.zones.zone_id,
                crdb_internal.zones.subzone_id,
                target,
                database_name,
                table_name,
                index_name,
                partition_violations.type,
                partition_violations.config,
                partition_violations.violation_start,
                partition_violations.violating_ranges
            FROM
                crdb_internal.zones, partition_violations
            WHERE
                crdb_internal.zones.zone_id
                = partition_violations.zone_id
        )
SELECT * FROM report;
```

14. Apply stricter replica placement settings
```
SET enable_multiregion_placement_policy=on;
```

15. Next, use the ALTER DATABASE ... PLACEMENT RESTRICTED statement to disable non-voting replicas for regional tables.
```
ALTER DATABASE django PLACEMENT RESTRICTED;
```

16. Now that you have restricted the placement of non-voting replicas for all regional tables, you can run another replication report to see the effects:
(This may take a couple of mins to have an affect.)
```
SELECT * FROM system.replication_constraint_stats WHERE violating_ranges > 0;
```

## Clean Up

Run the following command to cleanup.
```
kubectl delete -f ./kubernetes/deployment.yaml --context $clus1 --namespace $azregion
kubectl delete -f ./kubernetes/deployment.yaml --context $clus2 --namespace $awsregion
kubectl delete -f ./kubernetes/deployment.yaml --context $clus3 --namespace $gcpregion
```
Then exec back into our secure client and drop the database.
```
kubectl exec -it cockroachdb-client-secure \
-n $azregion \
--context $clus1 \
-- ./cockroach sql \
--certs-dir=/cockroach-certs \
--host=cockroachdb-public
```
Change the database to another database then drop `django` databases.
```
USE defaultdb;

DROP django;
```

---

Further testing could be as below to look at specific ranges an their placement within the cluster.
```
SHOW RANGE FROM TABLE cockroach_example_customers FOR ROW ('europe-west4','3cabc7f8-9f8a-4e18-ae17-17cf62db8ad0');

SHOW RANGE FROM TABLE cockroach_example_customers FOR ROW ('europe-west4','67972930-8582-4413-99df-91f68feb6297');

SHOW RANGE FROM TABLE cockroach_example_customers FOR ROW ('eu-west-1','a61cf64f-2bd5-4529-b773-557729c76480');
```




kubectl get nodes -o json | jq '.items[] | .status .addresses[] | select(.type=="ExternalIP") | .address'