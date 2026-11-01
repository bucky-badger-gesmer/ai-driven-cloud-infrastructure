# Verification log: SEIS 616 Software Delivery Pipeline

Account 762760349846, us-east-1, CLI profile `aaron@gesm4267`. Stack `seis616-pipeline` launched from the GitHub copy of `pipeline.yaml` at commit `192d80e` (SHA-256 `208ceab2…23ed3`).

## 1. Stack status and outputs

_2026-11-01 16:13:27 UTC_

```console
$ aws cloudformation describe-stacks --stack-name seis616-pipeline --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs[].[OutputKey,OutputValue]}' --output json
{
    "Status": "CREATE_COMPLETE",
    "Outputs": [
        [
            "BuildProjectName",
            "seis616-pipeline-build"
        ],
        [
            "ArtifactBucketName",
            "seis616-pipeline-artifactbucket-nekqzz1symsv"
        ],
        [
            "PipelineName",
            "seis616-pipeline"
        ],
        [
            "ApprovalTopicArn",
            "arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E"
        ],
        [
            "PipelineConsoleUrl",
            "https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/pipelines/seis616-pipeline/view"
        ]
    ]
}
```

## 2. Stack resources

_2026-11-01 16:13:27 UTC_

```console
$ aws cloudformation describe-stack-resources --stack-name seis616-pipeline --query 'StackResources[].[LogicalResourceId,ResourceType,ResourceStatus]' --output table
----------------------------------------------------------------------
|                       DescribeStackResources                       |
+----------------+-------------------------------+-------------------+
|  ApprovalTopic |  AWS::SNS::Topic              |  CREATE_COMPLETE  |
|  ArtifactBucket|  AWS::S3::Bucket              |  CREATE_COMPLETE  |
|  BuildProject  |  AWS::CodeBuild::Project      |  CREATE_COMPLETE  |
|  CodeBuildRole |  AWS::IAM::Role               |  CREATE_COMPLETE  |
|  Pipeline      |  AWS::CodePipeline::Pipeline  |  CREATE_COMPLETE  |
|  PipelineRole  |  AWS::IAM::Role               |  CREATE_COMPLETE  |
+----------------+-------------------------------+-------------------+
```

## 3. Pipeline definition: stages and actions

_2026-11-01 16:13:27 UTC_

```console
$ aws codepipeline get-pipeline --name seis616-pipeline --query 'pipeline.{type:pipelineType,stages:stages[].{stage:name,actions:actions[].[name,actionTypeId.category,actionTypeId.provider]},triggers:triggers}' --output json
{
    "type": "V2",
    "stages": [
        {
            "stage": "Source",
            "actions": [
                [
                    "Source",
                    "Source",
                    "CodeStarSourceConnection"
                ]
            ]
        },
        {
            "stage": "Approval",
            "actions": [
                [
                    "MaintainerReview",
                    "Approval",
                    "Manual"
                ]
            ]
        },
        {
            "stage": "Build",
            "actions": [
                [
                    "TestAndPackage",
                    "Build",
                    "CodeBuild"
                ]
            ]
        }
    ],
    "triggers": [
        {
            "providerType": "CodeStarSourceConnection",
            "gitConfiguration": {
                "sourceActionName": "Source",
                "push": [
                    {
                        "branches": {
                            "includes": [
                                "main"
                            ]
                        },
                        "filePaths": {
                            "excludes": [
                                "README.md",
                                "docs/**"
                            ]
                        }
                    }
                ]
            }
        }
    ]
}
```

## 4. SNS subscription before confirmation

_2026-11-01 16:13:28 UTC_

```console
$ aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E --query 'Subscriptions[].[Protocol,Endpoint,SubscriptionArn]' --output table
-----------------------------------------------------------
|                ListSubscriptionsByTopic                 |
+-------+-------------------------+-----------------------+
|  email|  <maintainer email>  |  PendingConfirmation  |
+-------+-------------------------+-----------------------+
```

## 5. Initial execution (created with the pipeline) paused at Approval

_2026-11-01 16:13:28 UTC_

```console
$ aws codepipeline get-pipeline-state --name seis616-pipeline --query 'stageStates[].{stage:stageName,status:latestExecution.status,executionId:latestExecution.pipelineExecutionId}' --output table
--------------------------------------------------------------------
|                         GetPipelineState                         |
+---------------------------------------+-----------+--------------+
|              executionId              |   stage   |   status     |
+---------------------------------------+-----------+--------------+
|  acb33a38-11aa-4a5c-95b3-7b1565ddfe27 |  Source   |  Succeeded   |
|  acb33a38-11aa-4a5c-95b3-7b1565ddfe27 |  Approval |  InProgress  |
|  None                                 |  Build    |  None        |
+---------------------------------------+-----------+--------------+
```

## 6. Approval action resolved configuration (source variables substituted)

_2026-11-01 16:13:35 UTC_

```console
$ aws codepipeline list-action-executions --pipeline-name seis616-pipeline --filter pipelineExecutionId=acb33a38-11aa-4a5c-95b3-7b1565ddfe27 --query 'actionExecutionDetails[?actionName==`MaintainerReview`].input.resolvedConfiguration' --output json
[
    {
        "CustomData": "Review commit 192d80e82423ad8035dc16330deca35b8bdfc0df: feat: add rectangle app, buildspec, and delivery pipeline template\n\nCo-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>",
        "ExternalEntityLink": "https://github.com/bucky-badger-gesmer/seis616-delivery-pipeline/commit/192d80e82423ad8035dc16330deca35b8bdfc0df",
        "NotificationArn": "arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E"
    }
]
```

## 7. Reject the initial execution

_2026-11-01 16:13:41 UTC_

```console
$ aws codepipeline put-approval-result --pipeline-name seis616-pipeline --stage-name Approval --action-name MaintainerReview --token 52a4f44e-e53b-4800-b909-d8985e67847d --result summary='Rejected for verification: reject path test',status=Rejected --output json
{
    "approvedAt": "2026-11-01T10:13:41.546000-06:00"
}
```

## 8. After rejection: execution failed, Build never ran

_2026-11-01 16:13:47 UTC_

```console
$ aws codepipeline get-pipeline-state --name seis616-pipeline --query 'stageStates[].{stage:stageName,status:latestExecution.status,summary:actionStates[0].latestExecution.summary}' --output table; aws codepipeline get-pipeline-execution --pipeline-name seis616-pipeline --pipeline-execution-id acb33a38-11aa-4a5c-95b3-7b1565ddfe27 --query 'pipelineExecution.[status,statusSummary]' --output text; aws codebuild list-builds-for-project --project-name seis616-pipeline-build --output json
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                           GetPipelineState                                                                                           |
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|   stage  |  status    |                                                                                   summary                                                                                    |
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|  Source  |  Succeeded |  {"ProviderType":"GitHub","CommitMessage":"feat: add rectangle app, buildspec, and delivery pipeline template\n\nCo-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"}   |
|  Approval|  Failed    |  Rejected for verification: reject path test                                                                                                                                 |
|  Build   |  None      |  None                                                                                                                                                                        |
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Failed	None
{
    "ids": []
}
```

## 9. SNS subscription after the maintainer confirmed it

_2026-11-01 16:16:19 UTC_

```console
$ aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E --query 'Subscriptions[].[Protocol,Endpoint,SubscriptionArn]' --output table
-----------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                ListSubscriptionsByTopic                                                                 |
+-------+------------------------+------------------------------------------------------------------------------------------------------------------------+
|  email|  <maintainer email> |  arn:aws:sns:us-east-1:762760349846:seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E:b181444d-92db-416c-94c0-f929557afda7   |
+-------+------------------------+------------------------------------------------------------------------------------------------------------------------+
```

## 10. Push of 6fbc395 started a new execution, paused at Approval

_2026-11-01 16:16:57 UTC_

```console
$ aws codepipeline list-pipeline-executions --pipeline-name seis616-pipeline --query 'pipelineExecutionSummaries[].[pipelineExecutionId,status,trigger.triggerType,sourceRevisions[0].revisionId]' --output table; aws codepipeline get-pipeline-state --name seis616-pipeline --query 'stageStates[].{stage:stageName,status:latestExecution.status,executionId:latestExecution.pipelineExecutionId}' --output table
----------------------------------------------------------------------------------------------------------------------
|                                               ListPipelineExecutions                                               |
+---------------------------------------+-------------+-----------------+--------------------------------------------+
|  e460dcae-ee18-43e8-a252-b04d85fd83cb |  InProgress |  WebhookV2      |  6fbc395634461cc301252b8866b85a2ffcb6e274  |
|  acb33a38-11aa-4a5c-95b3-7b1565ddfe27 |  Failed     |  CreatePipeline |  192d80e82423ad8035dc16330deca35b8bdfc0df  |
+---------------------------------------+-------------+-----------------+--------------------------------------------+
--------------------------------------------------------------------
|                         GetPipelineState                         |
+---------------------------------------+-----------+--------------+
|              executionId              |   stage   |   status     |
+---------------------------------------+-----------+--------------+
|  e460dcae-ee18-43e8-a252-b04d85fd83cb |  Source   |  Succeeded   |
|  e460dcae-ee18-43e8-a252-b04d85fd83cb |  Approval |  InProgress  |
|  None                                 |  Build    |  None        |
+---------------------------------------+-----------+--------------+
```

## 11. Maintainer approved in the console; pipeline resumed and Build ran

_2026-11-01 16:18:17 UTC_

```console
$ aws codepipeline list-action-executions --pipeline-name seis616-pipeline --filter pipelineExecutionId=e460dcae-ee18-43e8-a252-b04d85fd83cb --query 'actionExecutionDetails[].{action:actionName,status:status,start:startTime,end:lastUpdateTime,by:updatedBy,summary:output.executionResult.externalExecutionSummary}' --output table; aws codepipeline get-pipeline-execution --pipeline-name seis616-pipeline --pipeline-execution-id e460dcae-ee18-43e8-a252-b04d85fd83cb --query 'pipelineExecution.[pipelineExecutionId,status]' --output text
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                                                  ListActionExecutions                                                                                                                                  |
+------------------+---------------------------------------+-----------------------------------+-----------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------------+
|      action      |                  by                   |                end                |               start               |  status    |                                                                summary                                                                 |
+------------------+---------------------------------------+-----------------------------------+-----------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------------+
|  TestAndPackage  |  None                                 |  2026-11-01T10:18:15.343000-06:00 |  2026-11-01T10:17:43.525000-06:00 |  Succeeded |  None                                                                                                                                  |
|  MaintainerReview|  arn:aws:iam::762760349846:user/aaron |  2026-11-01T10:17:42.976000-06:00 |  2026-11-01T10:16:51.921000-06:00 |  Succeeded |  Approved by arn:aws:iam::762760349846:user/aaron                                                                                      |
|  Source          |  None                                 |  2026-11-01T10:16:51.378000-06:00 |  2026-11-01T10:16:39.470000-06:00 |  Succeeded |  {"ProviderType":"GitHub","CommitMessage":"feat: add rectangle diagonal\n\nCo-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"}   |
+------------------+---------------------------------------+-----------------------------------+-----------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------------+
e460dcae-ee18-43e8-a252-b04d85fd83cb	Succeeded
```

## 12. CodeBuild build phases

_2026-11-01 16:18:25 UTC_

```console
$ aws codebuild batch-get-builds --ids seis616-pipeline-build:7410a910-49ff-43fd-b652-c7384d46e565 --query 'builds[0].{status:buildStatus,initiator:initiator,source:resolvedSourceVersion,phases:phases[].[phaseType,phaseStatus,durationInSeconds]}' --output json
{
    "status": "SUCCEEDED",
    "initiator": "codepipeline/seis616-pipeline",
    "source": "6fbc395634461cc301252b8866b85a2ffcb6e274",
    "phases": [
        [
            "SUBMITTED",
            "SUCCEEDED",
            0
        ],
        [
            "QUEUED",
            "SUCCEEDED",
            0
        ],
        [
            "PROVISIONING",
            "SUCCEEDED",
            4
        ],
        [
            "DOWNLOAD_SOURCE",
            "SUCCEEDED",
            2
        ],
        [
            "INSTALL",
            "SUCCEEDED",
            15
        ],
        [
            "PRE_BUILD",
            "SUCCEEDED",
            0
        ],
        [
            "BUILD",
            "SUCCEEDED",
            0
        ],
        [
            "POST_BUILD",
            "SUCCEEDED",
            0
        ],
        [
            "UPLOAD_ARTIFACTS",
            "SUCCEEDED",
            1
        ],
        [
            "FINALIZING",
            "SUCCEEDED",
            0
        ],
        [
            "COMPLETED",
            null,
            null
        ]
    ]
}
```

## 13. Build log excerpts (runtime, tests, artifact smoke run)

_2026-11-01 16:18:25 UTC_

```console
$ aws logs get-log-events --log-group-name /aws/codebuild/seis616-pipeline-build --log-stream-name 7410a910-49ff-43fd-b652-c7384d46e565 --query 'events[].message' --output text | tr '\t' '\n' | grep -E 'Python 3|python: 3|PASSED|FAILED|passed|area=|zipapp|Phase complete' 
[Container] 2026/11/01 16:17:51.659006 Phase complete: DOWNLOAD_SOURCE State: SUCCEEDED
[Container] 2026/11/01 16:18:07.084218 Phase complete: INSTALL State: SUCCEEDED
platform linux -- Python 3.12.15, pytest-8.4.2, pluggy-1.7.0 -- /root/.pyenv/versions/3.12.15/bin/python
tests/test_rectangle.py::test_area PASSED                                [ 20%]
tests/test_rectangle.py::test_perimeter PASSED                           [ 40%]
tests/test_rectangle.py::test_diagonal PASSED                            [ 60%]
tests/test_rectangle.py::test_zero_sides PASSED                          [ 80%]
tests/test_rectangle.py::test_negative_side_rejected PASSED              [100%]
============================== 5 passed in 0.02s ===============================
[Container] 2026/11/01 16:18:07.926632 Phase complete: PRE_BUILD State: SUCCEEDED
[Container] 2026/11/01 16:18:07.979009 Running command python -m zipapp build -m 'rectangle.__main__:main' -o dist/rectangle.pyz --compress
area=12.0 perimeter=14.0 diagonal=5.0
[Container] 2026/11/01 16:18:08.148440 Phase complete: BUILD State: SUCCEEDED
[Container] 2026/11/01 16:18:08.185346 Phase complete: POST_BUILD State: SUCCEEDED
[Container] 2026/11/01 16:18:08.489236 Phase complete: UPLOAD_ARTIFACTS State: SUCCEEDED
```

## 14. CodeBuild test report

_2026-11-01 16:18:27 UTC_

```console
$ aws codebuild batch-get-reports --report-arns arn:aws:codebuild:us-east-1:762760349846:report/seis616-pipeline-build-pytest-reports:7410a910-49ff-43fd-b652-c7384d46e565 --query 'reports[0].{status:status,summary:testSummary}' --output json; aws codebuild describe-test-cases --report-arn arn:aws:codebuild:us-east-1:762760349846:report/seis616-pipeline-build-pytest-reports:7410a910-49ff-43fd-b652-c7384d46e565 --query 'testCases[].[name,status]' --output table
{
    "status": "SUCCEEDED",
    "summary": {
        "total": 5,
        "statusCounts": {
            "ERROR": 0,
            "FAILED": 0,
            "SKIPPED": 0,
            "SUCCEEDED": 5,
            "UNKNOWN": 0
        },
        "durationInNanoSeconds": 19000000
    }
}
----------------------------------------------
|              DescribeTestCases             |
+------------------------------+-------------+
|  test_negative_side_rejected |  SUCCEEDED  |
|  test_area                   |  SUCCEEDED  |
|  test_zero_sides             |  SUCCEEDED  |
|  test_diagonal               |  SUCCEEDED  |
|  test_perimeter              |  SUCCEEDED  |
+------------------------------+-------------+
```

## 15. Pipeline artifacts in S3 (CodePipeline truncates artifact folder names to 10 characters)

_2026-11-01 16:18:48 UTC_

```console
$ aws s3 ls s3://seis616-pipeline-artifactbucket-nekqzz1symsv/ --recursive --human-readable
2026-11-01 10:18:09  896 Bytes seis616-pipeline/BuildOutpu/UCmUX5e
2026-11-01 10:16:51    4.7 KiB seis616-pipeline/SourceOutp/SQbQRrj
2026-11-01 10:12:42    4.7 KiB seis616-pipeline/SourceOutp/WWhQj6j
```

## 16. Download, unzip, and run the build artifact

_2026-11-01 16:18:49 UTC_

```console
$ cd /private/tmp/claude-501/-Users-aarongesmer-Documents-dev-kosmos/dd57f168-607e-4136-a0d1-ebaa33d9c690/scratchpad/artifact-dl && aws s3 cp s3://seis616-pipeline-artifactbucket-nekqzz1symsv/seis616-pipeline/BuildOutpu/UCmUX5e build-output.zip --only-show-errors && unzip -o -q build-output.zip && ls -l rectangle.pyz && python3 rectangle.pyz 3 4 && unzip -l rectangle.pyz
-rw-r--r--@ 1 aarongesmer  wheel  976 Nov  1 10:18 rectangle.pyz
area=12.0 perimeter=14.0 diagonal=5.0
Archive:  rectangle.pyz
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  11-01-2026 16:18   rectangle/
      571  11-01-2026 16:18   rectangle/__init__.py
      401  11-01-2026 16:18   rectangle/__main__.py
       76  11-01-2026 16:18   __main__.py
---------                     -------
     1048                     4 files
```

## 17. SNS delivery metrics for the approval topic

_2026-11-01 16:18:59 UTC_

```console
$ for m in NumberOfMessagesPublished NumberOfNotificationsDelivered NumberOfNotificationsFailed; do printf '%s: ' $m; aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name $m --dimensions Name=TopicName,Value=seis616-pipeline-ApprovalTopic-DPXPKb5TLC7E --start-time 2026-11-01T16:00:00Z --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 3600 --statistics Sum --query 'Datapoints[].Sum' --output text; done
NumberOfMessagesPublished: 2.0
NumberOfNotificationsDelivered: 2.0
NumberOfNotificationsFailed: 0.0
```

## 18. README-only push (6db9308) did not start an execution (trigger excludes README.md)

_2026-11-01 16:21:20 UTC_

```console
$ git -C /Users/aarongesmer/Documents/dev/kosmos/repositories/seis616-delivery-pipeline show --stat --format='%h %s (%cI)' 6db9308; aws codepipeline list-pipeline-executions --pipeline-name seis616-pipeline --query 'pipelineExecutionSummaries[].[pipelineExecutionId,status,trigger.triggerType,sourceRevisions[0].revisionId,startTime]' --output table
6db9308 docs: add pipeline overview and verification report (2026-11-01T10:19:57-06:00)

 README.md | 183 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 183 insertions(+)
---------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                ListPipelineExecutions                                                                 |
+---------------------------------------+------------+-----------------+--------------------------------------------+-----------------------------------+
|  e460dcae-ee18-43e8-a252-b04d85fd83cb |  Succeeded |  WebhookV2      |  6fbc395634461cc301252b8866b85a2ffcb6e274  |  2026-11-01T10:16:39.295000-06:00 |
|  acb33a38-11aa-4a5c-95b3-7b1565ddfe27 |  Failed    |  CreatePipeline |  192d80e82423ad8035dc16330deca35b8bdfc0df  |  2026-11-01T10:12:39.754000-06:00 |
+---------------------------------------+------------+-----------------+--------------------------------------------+-----------------------------------+
```
