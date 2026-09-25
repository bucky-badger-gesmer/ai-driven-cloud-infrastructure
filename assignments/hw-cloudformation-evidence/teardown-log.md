# CorpWeb teardown log — 2026-09-25T19:52:24Z

$ aws sts get-caller-identity --profile aaron@gesm4267 --region us-east-1 --query '[Account,Arn]' --output text
762760349846	arn:aws:iam::762760349846:user/aaron

$ aws cloudformation delete-stack --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev

$ aws cloudformation wait stack-delete-complete --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev && echo stack-delete-complete: OK
stack-delete-complete: OK

# Post-delete checks — 2026-09-25T19:53:56Z

$ aws cloudformation describe-stacks --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev

aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id WebserversDev does not exist

$ aws cloudformation list-stacks --profile aaron@gesm4267 --region us-east-1 --stack-status-filter DELETE_COMPLETE --query 'StackSummaries[?StackName==`WebserversDev`]|[0].[StackName,StackStatus,DeletionTime]' --output text
WebserversDev	DELETE_COMPLETE	2026-09-25T19:52:24.981000+00:00

$ aws ec2 describe-instances --profile aaron@gesm4267 --region us-east-1 --instance-ids i-0ae4ac42201d170fc i-06563224368a9fe7b --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value,InstanceId,State.Name]' --output text
web2	i-06563224368a9fe7b	terminated
web1	i-0ae4ac42201d170fc	terminated

$ aws elbv2 describe-load-balancers --profile aaron@gesm4267 --region us-east-1 --names EngineeringLB

aws: [ERROR]: An error occurred (LoadBalancerNotFound) when calling the DescribeLoadBalancers operation: Load balancers '[EngineeringLB]' not found

$ aws elbv2 describe-target-groups --profile aaron@gesm4267 --region us-east-1 --names EngineeringWebservers

aws: [ERROR]: An error occurred (TargetGroupNotFound) when calling the DescribeTargetGroups operation: One or more target groups not found

$ aws ec2 describe-vpcs --profile aaron@gesm4267 --region us-east-1 --vpc-ids vpc-0c549166008e1b24c

aws: [ERROR]: An error occurred (InvalidVpcID.NotFound) when calling the DescribeVpcs operation: The vpc ID 'vpc-0c549166008e1b24c' does not exist

$ aws ec2 describe-security-groups --profile aaron@gesm4267 --region us-east-1 --filters Name=group-name,Values=WebserversSG --query 'SecurityGroups[].GroupId' --output text

$ aws iam get-role --profile aaron@gesm4267 --region us-east-1 --role-name WebserversDev-WebServerRole-CDZXPU8kBQb8

aws: [ERROR]: An error occurred (NoSuchEntity) when calling the GetRole operation: The role with name WebserversDev-WebServerRole-CDZXPU8kBQb8 cannot be found.

$ curl -s -m 5 -o /dev/null -w 'HTTP %{http_code} (curl exit follows)\n' http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/ ; echo curl exit: $?
HTTP 000 (curl exit follows)
curl exit: 6

