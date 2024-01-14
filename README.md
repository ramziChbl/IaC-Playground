# IaC-Playground

The following setup deploys the following instances:
1. orchestrator: Configuration management server and a bastion. It has a public IP
2. k8s-master: Where K8s will be installed, doesn't have a public gateway but still needs an Internet Gateway for internet access 

Spin up the infra:
```
cd cloudformation
export AWS_PROFILE=sysops-admin
export REGION=us-east-1
aws cloudformation deploy --stack-name common --template-file common.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name management --template-file management.yaml --parameter-overrides file://vars.json
aws cloudformation deploy --stack-name k8s-master --template-file k8s-master.yaml --parameter-overrides file://vars.json

# Associate k8s-master vpc to private hosted zone
hosted_zone_id=$(aws route53 list-hosted-zones --query "HostedZones[?starts_with(Name, \`infra.internal\`)].Id" --output text | cut -d'/' -f3)
vpc_id=$(aws ec2 describe-vpcs --filter "Name=tag:Name,Values=k8s-master" --query "Vpcs[*].VpcId" --output text)
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id "${hosted_zone_id}" --vpc "VPCRegion=${REGION},VPCId=${vpc_id}"

aws cloudformation deploy --stack-name vpc-peering --template-file vpc-peering.yaml --parameter-overrides file://vars.json
```

Tear it down:
```
aws cloudformation delete-stack --stack-name vpc-peering
aws cloudformation delete-stack --stack-name k8s-master
aws cloudformation delete-stack --stack-name management
aws cloudformation delete-stack --stack-name common
```

## IP Blocks and Routes

| VPC        | Subnet              | CIDR         | AZ         |
|------------|---------------------|--------------|------------|
| management | mgmt-default-subnet | 10.0.0.0/16  | us-east-1b |
| k8s        | k8s-public-az1      | 10.1.1.0/24  | us-east-1a |
| k8s        | k8s-public-az2      | 10.1.2.0/24  | us-east-1b |
| k8s        | k8s-private-az1     | 10.1.11.0/24 | us-east-1a |
| k8s        | k8s-private-az2     | 10.1.12.0/24 | us-east-1b |


| Route Table    | Destination | Target                        |
|----------------|-------------|-------------------------------|
| mgmt-rt        | 10.0.0.0/16 | Local                         |
|                | 10.1.0.0/16 | !Ref MgmtK8sPeeringConnection |
|                | 0.0.0.0/0   | !Ref MgmtInternetGateway      |
| k8s-nat-rt-az1 | 10.1.0.0/16 | Local                         |
|                | 10.0.0.0/16 | !Ref MgmtK8sPeeringConnection |
|                | 0.0.0.0/0   | !Ref K8sNatGatewayAz1         |
| k8s-nat-rt-az2 | 10.1.0.0/16 | Local                         |
|                | 10.0.0.0/16 | !Ref MgmtK8sPeeringConnection |
|                | 0.0.0.0/0   | !Ref K8sNatGatewayAz2         |



## TODO

* Add a [diagram](https://diagrams.mingrammer.com/) of the infrastructure
* Use salt reactors
* Leverage gitfs for salt states and pillars


## Useful Resources

* https://aws-ia.github.io/cfn-ps-aws-vpc/
* https://aws-ia-us-east-1.s3.us-east-1.amazonaws.com/cfn-ps-aws-vpc/templates/aws-vpc.template.yaml