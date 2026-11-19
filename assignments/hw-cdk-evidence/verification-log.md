# CDK Website HW: verification log

Run 2026-11-19 14:33:59 UTC, account 762760349846, us-east-1. Profile flags omitted below (`--profile aaron@gesm4267 --region us-east-1`).

## 1. Stacks

```
$ aws cloudformation describe-stacks --profile aaron@gesm4267 --region us-east-1 --query 'Stacks[?starts_with(StackName,`Website`)].[StackName,StackStatus]' --output table
--------------------------------------------
|              DescribeStacks              |
+----------------------+-------------------+
|  WebsiteServerStack  |  CREATE_COMPLETE  |
|  WebsiteNetworkStack |  CREATE_COMPLETE  |
+----------------------+-------------------+
```

## 2. Subnets: one public + one private in each of two AZs

```
$ aws ec2 describe-subnets --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0099fedebcd260c5a --query 'sort_by(Subnets,&CidrBlock)[].{Name:Tags[?Key==`Name`]|[0].Value,AZ:AvailabilityZone,CIDR:CidrBlock,PublicIP:MapPublicIpOnLaunch,Id:SubnetId}' --output table
------------------------------------------------------------------------------------------------------------------------
|                                                    DescribeSubnets                                                   |
+------------+--------------+---------------------------+-------------------------------------------------+------------+
|     AZ     |    CIDR      |            Id             |                      Name                       | PublicIP   |
+------------+--------------+---------------------------+-------------------------------------------------+------------+
|  us-east-1a|  10.0.0.0/24 |  subnet-01e23201ecf0747c3 |  WebsiteNetworkStack/WebsiteVpc/PublicSubnet1   |  True      |
|  us-east-1b|  10.0.1.0/24 |  subnet-0b135ed14a5bdf2e1 |  WebsiteNetworkStack/WebsiteVpc/PublicSubnet2   |  True      |
|  us-east-1a|  10.0.2.0/24 |  subnet-0d12dcab62a3be891 |  WebsiteNetworkStack/WebsiteVpc/PrivateSubnet1  |  False     |
|  us-east-1b|  10.0.3.0/24 |  subnet-0967837a04ae7dbad |  WebsiteNetworkStack/WebsiteVpc/PrivateSubnet2  |  False     |
+------------+--------------+---------------------------+-------------------------------------------------+------------+
```

## 3. Route tables: public subnets route to an internet gateway, private subnets are local-only

```
$ aws ec2 describe-route-tables --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0099fedebcd260c5a --query 'RouteTables[?Associations[?SubnetId]].{Subnet:Associations[0].SubnetId,Routes:join(`, `,Routes[].join(` -> `,[DestinationCidrBlock,GatewayId||`?`]))}' --output table
----------------------------------------------------------------------------------------
|                                  DescribeRouteTables                                 |
+---------------------------------------------------------+----------------------------+
|                         Routes                          |          Subnet            |
+---------------------------------------------------------+----------------------------+
|  10.0.0.0/16-> local, 0.0.0.0/0-> igw-04633c9068ed87822 |  subnet-0b135ed14a5bdf2e1  |
|  10.0.0.0/16-> local, 0.0.0.0/0-> igw-04633c9068ed87822 |  subnet-01e23201ecf0747c3  |
|  10.0.0.0/16-> local                                    |  subnet-0d12dcab62a3be891  |
|  10.0.0.0/16-> local                                    |  subnet-0967837a04ae7dbad  |
+---------------------------------------------------------+----------------------------+
```

```
$ aws ec2 describe-nat-gateways --profile aaron@gesm4267 --region us-east-1 --filter Name=vpc-id,Values=vpc-0099fedebcd260c5a --query 'length(NatGateways)'
0
```

## 4. Web servers: one in each public subnet

```
$ aws ec2 describe-instances --profile aaron@gesm4267 --region us-east-1 --instance-ids i-08b530cf0410b42c9 i-0ef795615f6066d60 --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,Id:InstanceId,State:State.Name,AZ:Placement.AvailabilityZone,Subnet:SubnetId,PublicIp:PublicIpAddress,SG:SecurityGroups[0].GroupName}' --output table
---------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                      DescribeInstances                                                                      |
+------------+----------------------+-------+-----------------+-------------------------------------------------------+----------+----------------------------+
|     AZ     |         Id           | Name  |    PublicIp     |                          SG                           |  State   |          Subnet            |
+------------+----------------------+-------+-----------------+-------------------------------------------------------+----------+----------------------------+
|  us-east-1b|  i-0ef795615f6066d60 |  web2 |  54.227.120.208 |  WebsiteServerStack-WebServerSG4D0F372C-BE5bXUpMmWlx  |  running |  subnet-0b135ed14a5bdf2e1  |
|  us-east-1a|  i-08b530cf0410b42c9 |  web1 |  100.48.195.231 |  WebsiteServerStack-WebServerSG4D0F372C-BE5bXUpMmWlx  |  running |  subnet-01e23201ecf0747c3  |
+------------+----------------------+-------+-----------------+-------------------------------------------------------+----------+----------------------------+
```

## 5. Security groups

```
$ aws ec2 describe-security-groups --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0099fedebcd260c5a --query 'SecurityGroups[?GroupName!=`default`].{Id:GroupId,Description:Description,Ingress:IpPermissions[].{Port:FromPort,FromCidr:IpRanges[].CidrIp,FromSG:UserIdGroupPairs[].GroupId}}' --output json
[
    {
        "Id": "sg-0b232d5cafd36aeaa",
        "Description": "Web servers: HTTP from anywhere",
        "Ingress": [
            {
                "Port": 80,
                "FromCidr": [
                    "0.0.0.0/0"
                ],
                "FromSG": []
            }
        ]
    },
    {
        "Id": "sg-0daf61e6ba1772663",
        "Description": "RDS: MySQL from the web servers only",
        "Ingress": [
            {
                "Port": 3306,
                "FromCidr": [],
                "FromSG": [
                    "sg-0b232d5cafd36aeaa"
                ]
            }
        ]
    },
    {
        "Id": "sg-0a6cf67603d023462",
        "Description": "ALB: HTTP from anywhere",
        "Ingress": [
            {
                "Port": 80,
                "FromCidr": [
                    "0.0.0.0/0"
                ],
                "FromSG": []
            }
        ]
    }
]
```

## 6. RDS: MySQL, private, subnet group = both private subnets

```
$ aws rds describe-db-instances --profile aaron@gesm4267 --region us-east-1 --query 'DBInstances[?Endpoint.Address==`websiteserverstack-databaseb269d8bb-rsxq10ez84e3.c6to8euw2plx.us-east-1.rds.amazonaws.com`].{Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Status:DBInstanceStatus,Public:PubliclyAccessible,SGs:VpcSecurityGroups[].VpcSecurityGroupId,SubnetGroup:DBSubnetGroup.DBSubnetGroupName,Subnets:DBSubnetGroup.Subnets[].[SubnetIdentifier,SubnetAvailabilityZone.Name]}' --output json
[
    {
        "Engine": "mysql",
        "Version": "8.4.11",
        "Class": "db.t3.micro",
        "Status": "available",
        "Public": false,
        "SGs": [
            "sg-0daf61e6ba1772663"
        ],
        "SubnetGroup": "websiteserverstack-databasesubnetgroup7d60f180-6hx8ui9ks4kl",
        "Subnets": [
            [
                "subnet-0967837a04ae7dbad",
                "us-east-1b"
            ],
            [
                "subnet-0d12dcab62a3be891",
                "us-east-1a"
            ]
        ]
    }
]
```

## 7. Load balancer: both targets healthy

```
$ aws elbv2 describe-target-health --profile aaron@gesm4267 --region us-east-1 --target-group-arn arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/Websit-LoadB-UHHKQEWBS9LK/f7d46365809838b5 --query 'TargetHealthDescriptions[].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State}' --output table
--------------------------------------------
|           DescribeTargetHealth           |
+------+-----------+-----------------------+
| Port |   State   |        Target         |
+------+-----------+-----------------------+
|  80  |  healthy  |  i-0ef795615f6066d60  |
|  80  |  healthy  |  i-08b530cf0410b42c9  |
+------+-----------+-----------------------+
```

## 8. Load balancer distributes requests across both web servers

```
$ for i in $(seq 10); do curl -s http://Websit-LoadB-IxNhmA8zrvT6-1491491979.us-east-1.elb.amazonaws.com/index.php; echo; done
Hi, I'm instance i-0ef795615f6066d60

Hi, I'm instance i-08b530cf0410b42c9

Hi, I'm instance i-08b530cf0410b42c9

Hi, I'm instance i-08b530cf0410b42c9

Hi, I'm instance i-0ef795615f6066d60

Hi, I'm instance i-0ef795615f6066d60

Hi, I'm instance i-08b530cf0410b42c9

Hi, I'm instance i-0ef795615f6066d60

Hi, I'm instance i-08b530cf0410b42c9

Hi, I'm instance i-08b530cf0410b42c9

```

## 9. Database reachable on 3306 from a web server (via SSM Run Command)

```
$ aws ssm get-command-invocation --profile aaron@gesm4267 --region us-east-1 --command-id e7b8ff3b-f036-4719-b425-22e6565e8052 --instance-id i-08b530cf0410b42c9 --query '{Status:Status,Output:StandardOutputContent}' --output json
{
    "Status": "Success",
    "Output": "10.0.3.209      websiteserverstack-databaseb269d8bb-rsxq10ez84e3.c6to8euw2plx.us-east-1.rds.amazonaws.com\nTCP 3306 OPEN from web1\n"
}
```

## 10. Database not reachable from outside the VPC

```
$ dig +short websiteserverstack-databaseb269d8bb-rsxq10ez84e3.c6to8euw2plx.us-east-1.rds.amazonaws.com; timeout 5 bash -c '</dev/tcp/websiteserverstack-databaseb269d8bb-rsxq10ez84e3.c6to8euw2plx.us-east-1.rds.amazonaws.com/3306' && echo 'TCP 3306 OPEN from workstation' || echo 'TCP 3306 FAILED from workstation (expected)'
10.0.3.209
TCP 3306 FAILED from workstation (expected)
```
