# route53-failover.md
# Purpose: Runbook commands for DNS failover (Route 53) and database failover (RDS promote read replica) during a disaster.
# Maps to: Day-56 (Cluster Backup Strategies), Day-57 (Cross Region Replication), Day-58 (Failover Procedures), Day-59 (Disaster Recovery Practice)

## Route 53 Failover

Switch the weighted/failover record value to the secondary (e.g. `secondary-cluster-xxx.elb.amazonaws.com`):

```bash
# Record the current endpoint (read-only check)
aws route53 list-resource-record-sets --hosted-zone-id ZONE_ID \
  --query "ResourceRecordSets[?Name=='app.example.com.']"

# Perform the update (replace the Value field with the secondary DNS name)
aws route53 change-resource-record-sets --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "secondary-cluster-xxx.elb.amazonaws.com"}]
      }
    }]
  }'

# Verify the change propagated
aws route53 get-change --id CHANGE_ID

# Validate DNS answers after propagation
dig +short app.example.com
```

For health-check / failover routing policies, briefly flip the `HealthCheckId` to point at the secondary region health check in the record set above.

## RDS Promote Read Replica

```bash
# Identify the replica (it must be a Multi-AZ read replica of the primary)
aws rds describe-db-clusters \
  --db-cluster-identifier primary-cluster --region us-east-1

# Promote the cross-region read replica to a standalone primary
aws rds promote-read-replica \
  --db-instance-identifier my-replica \
  --region us-west-2

# Wait until the promoted instance reaches the "available" status
aws rds wait db-instance-available \
  --db-instance-identifier my-replica --region us-west-2

# Retrieve the new endpoint to update the application / DNS
aws rds describe-db-instances \
  --db-instance-identifier my-replica --region us-west-2 \
  --query "DBInstances[0].Endpoint.Address"
```

> Note: `promote-read-replica` is used for single-instance read replicas; for Aurora use `aws rds promote-read-replica-db-cluster`. Update the DB endpoint above (or the app's connection string / Secret) to point at the promoted instance before failing traffic back.