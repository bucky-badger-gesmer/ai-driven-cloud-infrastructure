# Teardown log: SEIS 616 Software Delivery Pipeline

Account 762760349846, us-east-1, CLI profile `aaron@gesm4267`.

## 1. Empty the artifact bucket

_2026-11-01 16:23:38 UTC_

```console
$ aws s3 rm s3://seis616-pipeline-artifactbucket-nekqzz1symsv --recursive && aws s3api list-object-versions --bucket seis616-pipeline-artifactbucket-nekqzz1symsv --query '{versions:length(Versions||`[]`),markers:length(DeleteMarkers||`[]`)}' --output json
delete: s3://seis616-pipeline-artifactbucket-nekqzz1symsv/seis616-pipeline/BuildOutpu/UCmUX5e
delete: s3://seis616-pipeline-artifactbucket-nekqzz1symsv/seis616-pipeline/SourceOutp/SQbQRrj
delete: s3://seis616-pipeline-artifactbucket-nekqzz1symsv/seis616-pipeline/SourceOutp/WWhQj6j
{
    "versions": 0,
    "markers": 0
}
```

## 2. Delete the stack

_2026-11-01 16:25:11 UTC_

```console
$ aws cloudformation delete-stack --stack-name seis616-pipeline && aws cloudformation wait stack-delete-complete --stack-name seis616-pipeline && aws cloudformation describe-stacks --stack-name seis616-pipeline 2>&1 | tail -1; aws cloudformation list-stacks --stack-status-filter DELETE_COMPLETE --query 'StackSummaries[?StackName==`seis616-pipeline`]|[0].[StackName,StackStatus,DeletionTime]' --output text
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id seis616-pipeline does not exist
seis616-pipeline	DELETE_COMPLETE	2026-11-01T16:23:39.388000+00:00
```

## 3. Delete leftovers CodeBuild created outside the stack (pipeline build)

_2026-11-01 16:25:28 UTC_

```console
$ aws codebuild delete-report-group --arn arn:aws:codebuild:us-east-1:762760349846:report-group/seis616-pipeline-build-pytest-reports --delete-reports && aws logs delete-log-group --log-group-name /aws/codebuild/seis616-pipeline-build && echo deleted
deleted
```

## 4. Delete the console test project gesmer-java-project

_2026-11-01 16:25:31 UTC_

```console
$ aws codebuild delete-project --name gesmer-java-project && aws codebuild delete-report-group --arn arn:aws:codebuild:us-east-1:762760349846:report-group/gesmer-java-project-JUNITXML-AutoDiscovered --delete-reports && aws logs delete-log-group --log-group-name /aws/codebuild/gesmer-java-project && aws s3 rm s3://cf-templates-tx49p2b8cadw-us-east-1/rectangle-2611011538.zip && aws s3 rm s3://cf-templates-tx49p2b8cadw-us-east-1/rectangle-2611011540.zip
delete: s3://cf-templates-tx49p2b8cadw-us-east-1/rectangle-2611011538.zip
delete: s3://cf-templates-tx49p2b8cadw-us-east-1/rectangle-2611011540.zip
```

## 5. Delete the console project's service role and its two managed policies

_2026-11-01 16:25:35 UTC_

```console
$ for p in $(aws iam list-attached-role-policies --role-name codebuild-gesmer-java-project-service-role --query 'AttachedPolicies[].PolicyArn' --output text); do aws iam detach-role-policy --role-name codebuild-gesmer-java-project-service-role --policy-arn $p; for v in $(aws iam list-policy-versions --policy-arn $p --query 'Versions[?!IsDefaultVersion].VersionId' --output text); do aws iam delete-policy-version --policy-arn $p --version-id $v; done; aws iam delete-policy --policy-arn $p && echo deleted $p; done; aws iam list-role-policies --role-name codebuild-gesmer-java-project-service-role --query PolicyNames --output text; aws iam delete-role --role-name codebuild-gesmer-java-project-service-role && echo deleted role codebuild-gesmer-java-project-service-role
deleted arn:aws:iam::762760349846:policy/service-role/CodeBuildBasePolicy-gesmer-java-project-us-east-1
deleted arn:aws:iam::762760349846:policy/service-role/CodeBuildCodeConnectionsSourceCredentialsPolicy-gesmer-java-project-us-east-1-762760349846
deleted role codebuild-gesmer-java-project-service-role
```

## 6. Final sweep: nothing left from this assignment

_2026-11-01 16:25:50 UTC_

```console
$ echo '-- live stacks:'; aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE --query 'StackSummaries[].StackName' --output text; echo '-- pipelines:'; aws codepipeline list-pipelines --query 'pipelines[].name' --output text; echo '-- codebuild projects:'; aws codebuild list-projects --query projects --output text; echo '-- report groups:'; aws codebuild list-report-groups --query reportGroups --output text; echo '-- /aws/codebuild log groups:'; aws logs describe-log-groups --log-group-name-prefix /aws/codebuild/ --query 'logGroups[].logGroupName' --output text; echo '-- buckets:'; aws s3 ls; echo '-- cf-templates bucket contents:'; aws s3 ls s3://cf-templates-tx49p2b8cadw-us-east-1/; echo '-- SNS topics:'; aws sns list-topics --query 'Topics[].TopicArn' --output text; echo '-- SNS subscriptions:'; aws sns list-subscriptions --query 'Subscriptions[].[Endpoint,TopicArn]' --output text; echo '-- IAM roles matching pipeline/codebuild:'; aws iam list-roles --query 'Roles[?contains(RoleName,`seis616`)||contains(RoleName,`codebuild`)||contains(RoleName,`gesmer-java`)].RoleName' --output text; echo '-- IAM policies matching gesmer-java:'; aws iam list-policies --scope Local --query 'Policies[?contains(PolicyName,`gesmer-java`)].PolicyName' --output text; echo '-- GitHub webhooks on java-project:'; gh api repos/bucky-badger-gesmer/java-project/hooks --jq 'length'; echo '-- GitHub connection (kept):'; aws codeconnections list-connections --query 'Connections[].[ConnectionName,ConnectionStatus]' --output text
-- live stacks:
-- pipelines:
-- codebuild projects:
-- report groups:
arn:aws:codebuild:us-east-1:762760349846:report-group/gesmer-java-project-JUNITXML-AutoDiscovered	arn:aws:codebuild:us-east-1:762760349846:report-group/seis616-pipeline-build-pytest-reports
-- /aws/codebuild log groups:
-- buckets:
2026-10-03 21:44:38 cf-templates-tx49p2b8cadw-us-east-1
-- cf-templates bucket contents:
2026-10-03 21:44:39       9192 2026-10-04T024438.091Z668-corpweb.json
-- SNS topics:
-- SNS subscriptions:
<maintainer email>	arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E
-- IAM roles matching pipeline/codebuild:
-- IAM policies matching gesmer-java:
-- GitHub webhooks on java-project:
0
-- GitHub connection (kept):
bucky-badger-gesmer-github	AVAILABLE
```

## 7. Remove the email subscription left on the deleted topic

_2026-11-01 16:26:04 UTC_

```console
$ aws sns unsubscribe --subscription-arn arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E:b181444d-92db-416c-94c0-f929557afda7 && aws sns list-subscriptions --query 'length(Subscriptions)' --output text
0
0
```

## 8. Report groups finished deleting

_2026-11-01 16:26:05 UTC_

```console
$ aws codebuild list-report-groups --query 'length(reportGroups)' --output text
0
```
