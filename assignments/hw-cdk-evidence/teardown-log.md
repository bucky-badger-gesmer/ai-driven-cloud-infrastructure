# CDK Website HW: teardown log

`npx cdk destroy --all --force` at 2026-11-19 14:39 UTC:

```
WebsiteServerStack: destroying... [1/2]
✅  WebsiteServerStack: destroyed
WebsiteNetworkStack: destroying... [2/2]
✅  WebsiteNetworkStack: destroyed
```

Sweep (profile aaron@gesm4267, us-east-1):

```
Website* stacks (any live status)  
CDKToolkit (kept)                  CREATE_COMPLETE 
non-terminated instances           0 
VPC vpc-0099fedebcd260c5a          InvalidVpcID.NotFound 
RDS instances                      0 
RDS snapshots                      0 
RDS automated backups              0 
RDS subnet groups (website*)       0 
load balancers                     0 
target groups                      0 
secrets (incl. pending deletion)   
Elastic IPs                        0 
log groups (Website*/rds)          1 
IAM roles Website*                 0 
SSM managed instances              0 
```

The one leftover log group was `/aws/lambda/WebsiteNetworkStack-CustomVpcRestrictDefaultSGCust-hO9KJGVRYHBN` (0 bytes). It belongs to the Lambda behind CDK's `restrictDefaultSecurityGroup` custom resource, and `cdk destroy` does not remove log groups. Deleted it:

```
$ aws logs delete-log-group --log-group-name /aws/lambda/WebsiteNetworkStack-CustomVpcRestrictDefaultSGCust-hO9KJGVRYHBN
$ aws logs describe-log-groups --query 'length(logGroups[?contains(logGroupName,`Website`)||contains(logGroupName,`/rds/`)])'
0
```

The generated DB secret does not appear even with `--include-planned-deletion`, so it was removed with the stack. Kept: the `CDKToolkit` bootstrap stack (needed for any future CDK deploy in this account/region).
