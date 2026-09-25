# CorpWeb verification log — 2026-09-25T19:49:52Z

$ aws sts get-caller-identity --profile aaron@gesm4267 --region us-east-1 --query '[Account,Arn]' --output text
762760349846	arn:aws:iam::762760349846:user/aaron

$ aws cloudformation describe-stacks --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev --query 'Stacks[0].{Status:StackStatus,Created:CreationTime,Parameters:Parameters,Outputs:Outputs,Capabilities:Capabilities}' --output json
{
    "Status": "CREATE_COMPLETE",
    "Created": "2026-09-25T19:46:03.457000+00:00",
    "Parameters": [
        {
            "ParameterKey": "KeyPair",
            "ParameterValue": "lab-key"
        },
        {
            "ParameterKey": "YourIp",
            "ParameterValue": "x.x.x.x/32"
        },
        {
            "ParameterKey": "InstanceType",
            "ParameterValue": "t2.micro"
        }
    ],
    "Outputs": [
        {
            "OutputKey": "WebUrl",
            "OutputValue": "EngineeringLB-1369476053.us-east-1.elb.amazonaws.com",
            "Description": "DNS name of the EngineeringLB load balancer"
        }
    ],
    "Capabilities": [
        "CAPABILITY_IAM"
    ]
}

$ aws cloudformation list-stack-resources --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev --query 'StackResourceSummaries[].[LogicalResourceId,ResourceType,ResourceStatus,PhysicalResourceId]' --output table
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                    ListStackResources                                                                                                    |
+------------------------------------+--------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------+
|  EngineeringLB                     |  AWS::ElasticLoadBalancingV2::LoadBalancer |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:loadbalancer/app/EngineeringLB/aa94b21d4d6eff99                |
|  EngineeringLBListener             |  AWS::ElasticLoadBalancingV2::Listener     |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:listener/app/EngineeringLB/aa94b21d4d6eff99/51bd4c7d8bebd056   |
|  EngineeringVpc                    |  AWS::EC2::VPC                             |  CREATE_COMPLETE |  vpc-0c549166008e1b24c                                                                                              |
|  EngineeringWebservers             |  AWS::ElasticLoadBalancingV2::TargetGroup  |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76             |
|  InternetGateway                   |  AWS::EC2::InternetGateway                 |  CREATE_COMPLETE |  igw-0514be4c741be0be3                                                                                              |
|  PublicRoute                       |  AWS::EC2::Route                           |  CREATE_COMPLETE |  rtb-036b234ff38dee310|0.0.0.0/0                                                                                    |
|  PublicRouteTable                  |  AWS::EC2::RouteTable                      |  CREATE_COMPLETE |  rtb-036b234ff38dee310                                                                                              |
|  PublicSubnet1                     |  AWS::EC2::Subnet                          |  CREATE_COMPLETE |  subnet-034171f3d538fe8d9                                                                                           |
|  PublicSubnet1RouteTableAssociation|  AWS::EC2::SubnetRouteTableAssociation     |  CREATE_COMPLETE |  rtbassoc-0e698a14ee48a6584                                                                                         |
|  PublicSubnet2                     |  AWS::EC2::Subnet                          |  CREATE_COMPLETE |  subnet-0fd56fbd357763fe5                                                                                           |
|  PublicSubnet2RouteTableAssociation|  AWS::EC2::SubnetRouteTableAssociation     |  CREATE_COMPLETE |  rtbassoc-00c8272fef15019fd                                                                                         |
|  VpcGatewayAttachment              |  AWS::EC2::VPCGatewayAttachment            |  CREATE_COMPLETE |  IGW|vpc-0c549166008e1b24c                                                                                          |
|  WebServerInstanceProfile          |  AWS::IAM::InstanceProfile                 |  CREATE_COMPLETE |  WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML                                                                |
|  WebServerLaunchTemplate           |  AWS::EC2::LaunchTemplate                  |  CREATE_COMPLETE |  lt-042d9d72b0cb235dd                                                                                               |
|  WebServerRole                     |  AWS::IAM::Role                            |  CREATE_COMPLETE |  WebserversDev-WebServerRole-CDZXPU8kBQb8                                                                           |
|  WebserversSG                      |  AWS::EC2::SecurityGroup                   |  CREATE_COMPLETE |  sg-0b780a161fed6f4af                                                                                               |
|  web1                              |  AWS::EC2::Instance                        |  CREATE_COMPLETE |  i-0ae4ac42201d170fc                                                                                                |
|  web2                              |  AWS::EC2::Instance                        |  CREATE_COMPLETE |  i-06563224368a9fe7b                                                                                                |
+------------------------------------+--------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------+

$ aws ec2 describe-vpcs --profile aaron@gesm4267 --region us-east-1 --vpc-ids vpc-0c549166008e1b24c --query 'Vpcs[].{VpcId:VpcId,Cidr:CidrBlock,Name:Tags[?Key==`aws:cloudformation:logical-id`]|[0].Value}' --output table
------------------------------------------------------------
|                       DescribeVpcs                       |
+-------------+------------------+-------------------------+
|    Cidr     |      Name        |          VpcId          |
+-------------+------------------+-------------------------+
|  10.0.0.0/18|  EngineeringVpc  |  vpc-0c549166008e1b24c  |
+-------------+------------------+-------------------------+

$ aws ec2 describe-subnets --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c --query 'Subnets[].{LogicalId:Tags[?Key==`aws:cloudformation:logical-id`]|[0].Value,SubnetId:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,PublicIpOnLaunch:MapPublicIpOnLaunch}' --output table
------------------------------------------------------------------------------------------------
|                                        DescribeSubnets                                       |
+------------+--------------+----------------+-------------------+-----------------------------+
|     AZ     |    Cidr      |   LogicalId    | PublicIpOnLaunch  |          SubnetId           |
+------------+--------------+----------------+-------------------+-----------------------------+
|  us-east-1b|  10.0.1.0/24 |  PublicSubnet2 |  True             |  subnet-0fd56fbd357763fe5   |
|  us-east-1a|  10.0.0.0/24 |  PublicSubnet1 |  True             |  subnet-034171f3d538fe8d9   |
+------------+--------------+----------------+-------------------+-----------------------------+

$ aws ec2 describe-route-tables --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c Name=tag:aws:cloudformation:logical-id,Values=PublicRouteTable --query 'RouteTables[].{Routes:Routes[].[DestinationCidrBlock,GatewayId,State],Subnets:Associations[].SubnetId}' --output json
[
    {
        "Routes": [
            [
                "10.0.0.0/18",
                "local",
                "active"
            ],
            [
                "0.0.0.0/0",
                "igw-0514be4c741be0be3",
                "active"
            ]
        ],
        "Subnets": [
            "subnet-0fd56fbd357763fe5",
            "subnet-034171f3d538fe8d9"
        ]
    }
]

$ aws ec2 describe-instances --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,InstanceId:InstanceId,Type:InstanceType,State:State.Name,Subnet:SubnetId,AZ:Placement.AvailabilityZone,ImageId:ImageId,KeyName:KeyName,PublicIp:PublicIpAddress,IMDSv2:MetadataOptions.HttpTokens,Profile:IamInstanceProfile.Arn}' --output table
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                                          DescribeInstances                                                                                                                         |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+
|     AZ     |  IMDSv2   |        ImageId         |     InstanceId       | KeyName  | Name  |                                             Profile                                              |    PublicIp     |  State   |          Subnet           |   Type     |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+
|  us-east-1b|  optional |  ami-0fef201115eefe936 |  i-06563224368a9fe7b |  lab-key |  web2 |  arn:aws:iam::762760349846:instance-profile/WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML  |  44.211.147.153 |  running |  subnet-0fd56fbd357763fe5 |  t2.micro  |
|  us-east-1a|  optional |  ami-0fef201115eefe936 |  i-0ae4ac42201d170fc |  lab-key |  web1 |  arn:aws:iam::762760349846:instance-profile/WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML  |  3.235.150.34   |  running |  subnet-034171f3d538fe8d9 |  t2.micro  |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+

$ aws ec2 describe-images --profile aaron@gesm4267 --region us-east-1 --image-ids ami-0fef201115eefe936 --query 'Images[].[ImageId,Name,Architecture]' --output text
ami-0fef201115eefe936	al2023-ami-2023.12.20260918.0-kernel-6.18-x86_64	x86_64

$ aws ec2 describe-security-groups --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c Name=group-name,Values=WebserversSG --query 'SecurityGroups[].{GroupName:GroupName,GroupId:GroupId,Ingress:IpPermissions[].{Port:FromPort,Proto:IpProtocol,From:IpRanges[].CidrIp}}' --output json
[
    {
        "GroupName": "WebserversSG",
        "GroupId": "sg-0b780a161fed6f4af",
        "Ingress": [
            {
                "Port": 80,
                "Proto": "tcp",
                "From": [
                    "0.0.0.0/0"
                ]
            },
            {
                "Port": 22,
                "Proto": "tcp",
                "From": [
                    "x.x.x.x/32"
                ]
            }
        ]
    }
]

$ aws elbv2 describe-load-balancers --profile aaron@gesm4267 --region us-east-1 --names EngineeringLB --query 'LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,Type:Type,Scheme:Scheme,State:State.Code,AZs:AvailabilityZones[].ZoneName,SGs:SecurityGroups}' --output json
[
    {
        "Name": "EngineeringLB",
        "DNS": "EngineeringLB-1369476053.us-east-1.elb.amazonaws.com",
        "Type": "application",
        "Scheme": "internet-facing",
        "State": "active",
        "AZs": [
            "us-east-1a",
            "us-east-1b"
        ],
        "SGs": [
            "sg-0b780a161fed6f4af"
        ]
    }
]

$ aws elbv2 describe-listeners --profile aaron@gesm4267 --region us-east-1 --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:762760349846:loadbalancer/app/EngineeringLB/aa94b21d4d6eff99 --query 'Listeners[].{Port:Port,Protocol:Protocol,Action:DefaultActions[0].Type,TargetGroup:DefaultActions[0].TargetGroupArn}' --output json
[
    {
        "Port": 80,
        "Protocol": "HTTP",
        "Action": "forward",
        "TargetGroup": "arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76"
    }
]

$ aws elbv2 describe-target-groups --profile aaron@gesm4267 --region us-east-1 --names EngineeringWebservers --query 'TargetGroups[].{Name:TargetGroupName,Protocol:Protocol,Port:Port,TargetType:TargetType,HC_Protocol:HealthCheckProtocol,HC_Port:HealthCheckPort,HC_Path:HealthCheckPath}' --output json
[
    {
        "Name": "EngineeringWebservers",
        "Protocol": "HTTP",
        "Port": 80,
        "TargetType": "instance",
        "HC_Protocol": "HTTP",
        "HC_Port": "80",
        "HC_Path": "/"
    }
]

$ aws elbv2 describe-target-health --profile aaron@gesm4267 --region us-east-1 --target-group-arn arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76 --query 'TargetHealthDescriptions[].[Target.Id,Target.Port,TargetHealth.State]' --output table
------------------------------------------
|          DescribeTargetHealth          |
+----------------------+-----+-----------+
|  i-06563224368a9fe7b |  80 |  healthy  |
|  i-0ae4ac42201d170fc |  80 |  healthy  |
+----------------------+-----+-----------+

$ curl -s -i -m 5 http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/
HTTP/1.1 200 OK
Date: Fri, 25 Sep 2026 19:50:00 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: Apache/2.4.68 (Amazon Linux)
X-Powered-By: PHP/8.5.10

Hi, I'm instance i-06563224368a9fe7b

$ for i in $(seq 10); do curl -s http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/index.php; done
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-0ae4ac42201d170fc

$ ... | sort | uniq -c   (distribution over 20 requests)
  11 Hi, I'm instance i-06563224368a9fe7b
   9 Hi, I'm instance i-0ae4ac42201d170fc
$ ssh -i ~/.ssh/lab-key.pem -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 -o BatchMode=yes ec2-user@3.235.150.34 'echo "connected as $(whoami) on $(hostname)"; cat /etc/os-release | grep PRETTY_NAME; echo -n "instance-id: "; TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60"); curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id; echo; echo -n "httpd: "; systemctl is-active httpd; ls -l /var/www/html/; curl -s http://localhost/index.php'
Warning: Permanently added '3.235.150.34' (ED25519) to the list of known hosts.
connected as ec2-user on ip-10-0-0-93.ec2.internal
PRETTY_NAME="Amazon Linux 2023.12.20260918"
instance-id: i-0ae4ac42201d170fc
httpd: active
total 4
-rw-r--r--. 1 root root 142 Oct 14  2018 index.php
Hi, I'm instance i-0ae4ac42201d170fc

$ ssh -i ~/.ssh/lab-key.pem -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 -o BatchMode=yes ec2-user@44.211.147.153 'echo "connected as $(whoami) on $(hostname)"; cat /etc/os-release | grep PRETTY_NAME; echo -n "instance-id: "; TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60"); curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id; echo; echo -n "httpd: "; systemctl is-active httpd; ls -l /var/www/html/; curl -s http://localhost/index.php'
Warning: Permanently added '44.211.147.153' (ED25519) to the list of known hosts.
connected as ec2-user on ip-10-0-1-106.ec2.internal
PRETTY_NAME="Amazon Linux 2023.12.20260918"
instance-id: i-06563224368a9fe7b
httpd: active
total 4
-rw-r--r--. 1 root root 142 Oct 14  2018 index.php
Hi, I'm instance i-06563224368a9fe7b

