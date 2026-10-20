# Day 45 – Configure NAT Gateway for Internet Access in a Private VPC

## 📌 Project Overview

In Day 45 of my 50 Days AWS Practice Lab Challenge, I configured a NAT Gateway to provide outbound internet access to resources in an existing private VPC.

The lab environment already contained a private VPC, a private subnet, and a private route table. AWS CLI was also configured before the lab began.

Instead of creating a new VPC or modifying the original private route table, I created an additional private route table, configured a default route through the NAT Gateway, and associated the new route table with the private subnet.

This lab demonstrates how AWS networking components work together to provide outbound internet access without assigning public IP addresses to private instances.

## 🎯 Objectives

* Work with an existing Amazon VPC.
* Understand public and private subnet routing.
* Configure an Internet Gateway.
* Allocate an Elastic IP address.
* Create a NAT Gateway in a public subnet.
* Create an additional private route table.
* Route private subnet internet-bound traffic through the NAT Gateway.
* Verify the route table configuration and connectivity.
* Understand AWS networking security and cost considerations.

## 🛠️ AWS Services Used

* **Amazon VPC:** Provides the virtual network.
* **Amazon EC2:** Provides instances for connectivity testing, if required.
* **Internet Gateway (IGW):** Enables internet connectivity for public subnet resources.
* **NAT Gateway:** Allows private subnet resources to initiate outbound internet connections.
* **Elastic IP:** Provides a public IPv4 address for the NAT Gateway.
* **VPC Route Tables:** Determine where network traffic is directed.
* **Security Groups and Network ACLs:** Control network traffic.

## 🏗️ Architecture

```text
                     Internet
                        |
                        v
                 Internet Gateway
                        |
                +-------------------+
                |    Public Subnet  |
                |                   |
                |    NAT Gateway    |
                |    Elastic IP     |
                +---------+---------+
                          |
                          v
                 New Private Route Table
                 0.0.0.0/0 -> NAT Gateway
                          |
                          v
                 +-------------------+
                 |   Private Subnet  |
                 |                   |
                 |   Private EC2     |
                 |   No Public IP    |
                 +-------------------+
```

### How Traffic Flows

1. A private EC2 instance initiates an outbound connection to the internet.
2. The private subnet route table directs the traffic to the NAT Gateway.
3. The NAT Gateway translates the source address for internet-bound traffic.
4. The Internet Gateway forwards the traffic to the internet.
5. Response traffic returns through the NAT Gateway to the originating private instance.

A NAT Gateway permits outbound-initiated connections and their response traffic. It does not provide unrestricted inbound access from the internet to private instances.

## 📋 Existing and New Resources

| Resource                       | Configuration                         |
| ------------------------------ | ------------------------------------- |
| VPC                            | Existing private VPC                  |
| Private subnet                 | Existing subnet                       |
| Original private route table   | Existing; retained                    |
| Additional private route table | Created during the lab                |
| Internet Gateway               | Existing or configured during the lab |
| Public subnet                  | Existing or configured during the lab |
| Elastic IP                     | Allocated for the NAT Gateway         |
| NAT Gateway                    | Created in the public subnet          |
| AWS CLI                        | Already configured                    |

The actual resource IDs depend on the lab environment.

## 1. Verify the AWS CLI Configuration

AWS CLI was already configured in the lab, so running `aws configure` again was unnecessary.

Check the current identity and region:

```bash
aws sts get-caller-identity

export AWS_REGION=$(aws configure get region)

echo "$AWS_REGION"
```

Verify that the required permissions and region are correct before continuing.

## 2. Identify the Existing VPC and Private Subnet

List VPCs:

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

List subnets:

```bash
aws ec2 describe-subnets \
  --query 'Subnets[].{SubnetId:SubnetId,VpcId:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Identify the VPC ID and private subnet ID supplied by the lab.

Set the IDs for subsequent commands:

```bash
export VPC_ID=<EXISTING_VPC_ID>
export PRIVATE_SUBNET_ID=<EXISTING_PRIVATE_SUBNET_ID>
```

Replace the placeholders with actual IDs before running the commands.

Check the existing private route table:

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[].{RouteTableId:RouteTableId,Associations:Associations[].SubnetId,Routes:Routes[].DestinationCidrBlock}' \
  --output json \
  --region "$AWS_REGION"
```

This helps identify the existing route tables and their subnet associations.

## 3. Configure the Internet Gateway

An Internet Gateway provides the internet connectivity required by the NAT Gateway.

Check whether the VPC already has an Internet Gateway attached:

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query 'InternetGateways[].InternetGatewayId' \
  --output table \
  --region "$AWS_REGION"
```

If an Internet Gateway is already attached, use its existing ID. Otherwise, create and attach one if the lab permits it.

Create an Internet Gateway:

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text \
  --region "$AWS_REGION")

echo "$IGW_ID"
```

Attach it to the existing VPC:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

**Note:** Do not create or attach a second Internet Gateway if one is already attached to the VPC.

## 4. Identify or Configure the Public Subnet

The NAT Gateway must be created in a public subnet.

A public subnet requires a route to the Internet Gateway. Its resources also need suitable public addressing and security configuration where direct internet access is required.

List the subnets in the VPC:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Set the ID of the public subnet used by the lab:

```bash
export PUBLIC_SUBNET_ID=<EXISTING_PUBLIC_SUBNET_ID>
```

Identify the route table associated with that subnet and verify that its default route points to the Internet Gateway.

If required, configure the public route table:

```bash
aws ec2 create-route \
  --route-table-id <PUBLIC_ROUTE_TABLE_ID> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <INTERNET_GATEWAY_ID> \
  --region "$AWS_REGION"
```

Use this command only if the route does not already exist. If the route exists but needs changing, inspect the current configuration before using `replace-route`.

## 5. Allocate an Elastic IP Address

An Elastic IP provides the public IPv4 address used by the NAT Gateway.

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query AllocationId \
  --output text \
  --region "$AWS_REGION")

echo "$EIP_ALLOC_ID"
```

Verify the allocation:

```bash
aws ec2 describe-addresses \
  --allocation-ids "$EIP_ALLOC_ID" \
  --region "$AWS_REGION"
```

## 6. Create the NAT Gateway

Create the NAT Gateway in the public subnet and associate the Elastic IP:

```bash
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --allocation-id "$EIP_ALLOC_ID" \
  --query 'NatGateway.NatGatewayId' \
  --output text \
  --region "$AWS_REGION")

echo "$NAT_GW_ID"
```

Tag the NAT Gateway:

```bash
aws ec2 create-tags \
  --resources "$NAT_GW_ID" \
  --tags Key=Name,Value=Day45-NAT-Gateway \
  --region "$AWS_REGION"
```

Wait until the NAT Gateway is available:

```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids "$NAT_GW_ID" \
  --region "$AWS_REGION"
```

Verify its state:

```bash
aws ec2 describe-nat-gateways \
  --nat-gateway-ids "$NAT_GW_ID" \
  --query 'NatGateways[].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId,Addresses:NatGatewayAddresses[].PublicIp}' \
  --output table \
  --region "$AWS_REGION"
```

Continue after the NAT Gateway state becomes `available`.

## 7. Create an Additional Private Route Table

The VPC already had a private route table. For this lab, I created a second private route table rather than reusing the original one.

Create the additional route table in the existing VPC:

```bash
NEW_PRIVATE_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query 'RouteTable.RouteTableId' \
  --output text \
  --region "$AWS_REGION")

echo "$NEW_PRIVATE_RT_ID"
```

Tag the route table:

```bash
aws ec2 create-tags \
  --resources "$NEW_PRIVATE_RT_ID" \
  --tags Key=Name,Value=Day45-Private-NAT-Route-Table \
  --region "$AWS_REGION"
```

This creates a route table but does not automatically associate it with a subnet.

## 8. Add a Default Route Through the NAT Gateway

Configure the new private route table to forward internet-bound IPv4 traffic to the NAT Gateway:

```bash
aws ec2 create-route \
  --route-table-id "$NEW_PRIVATE_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_GW_ID" \
  --region "$AWS_REGION"
```

The route means that traffic destined for addresses outside the VPC's local CIDR can be sent through the NAT Gateway, subject to the route table's other, more-specific routes.

## 9. Associate the New Route Table With the Private Subnet

Associate the newly created route table with the existing private subnet:

```bash
aws ec2 associate-route-table \
  --route-table-id "$NEW_PRIVATE_RT_ID" \
  --subnet-id "$PRIVATE_SUBNET_ID" \
  --region "$AWS_REGION"
```

Verify the association:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$NEW_PRIVATE_RT_ID" \
  --query 'RouteTables[].Associations' \
  --output json \
  --region "$AWS_REGION"
```

### Important Routing Concept

A subnet can have only one explicit route-table association at a time. Associating the new route table with the private subnet changes that subnet's effective route table association from its previous explicit route table, if it had one.

The original private route table remains in the VPC and can continue serving other subnets.

Creating another route table is not required for every NAT Gateway setup; an existing private route table could instead be updated. In this lab, the additional route table was created as part of the exercise.

## 10. Verify the Private Route Table

Check the routes:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$NEW_PRIVATE_RT_ID" \
  --query 'RouteTables[].Routes' \
  --output table \
  --region "$AWS_REGION"
```

The expected routes include:

| Destination | Target      |
| ----------- | ----------- |
| VPC CIDR    | Local       |
| `0.0.0.0/0` | NAT Gateway |

The VPC's local route is automatically created by AWS. The default route should point to the NAT Gateway.

Verify the NAT Gateway:

```bash
aws ec2 describe-nat-gateways \
  --nat-gateway-ids "$NAT_GW_ID" \
  --query 'NatGateways[].State' \
  --output text \
  --region "$AWS_REGION"
```

The expected state is `available`.

## 11. Test Outbound Internet Access

To verify the configuration end to end, use a test EC2 instance in the private subnet.

The instance should not need a public IP address. It needs an appropriate security group, network ACL rules, DNS configuration, and a method of access, such as AWS Systems Manager Session Manager, if configured.

From the private instance, run:

```bash
curl -I https://aws.amazon.com
```

You can also test name resolution:

```bash
nslookup aws.amazon.com
```

A successful HTTPS request confirms outbound connectivity to that endpoint. DNS resolution alone does not prove that internet traffic can reach the endpoint.

If required, compare the external IP reported by a trusted IP-check service with the NAT Gateway's Elastic IP.

## 12. IAM Permissions Considerations

The IAM principal running these commands needs permissions for the networking operations used in the lab. Depending on the existing environment, these can include:

* `ec2:DescribeVpcs`
* `ec2:DescribeSubnets`
* `ec2:DescribeRouteTables`
* `ec2:CreateInternetGateway`
* `ec2:AttachInternetGateway`
* `ec2:AllocateAddress`
* `ec2:CreateNatGateway`
* `ec2:CreateRouteTable`
* `ec2:CreateRoute`
* `ec2:AssociateRouteTable`
* `ec2:CreateTags`
* Related modification and cleanup permissions

This list is illustrative rather than a complete IAM policy. The required permissions depend on the operations and resources in your lab. Follow least-privilege access wherever possible.

## 13. Troubleshooting

| Issue                                      | Possible solution                                                                 |
| ------------------------------------------ | --------------------------------------------------------------------------------- |
| NAT Gateway is not available               | Check its state, subnet, Elastic IP, and AWS service errors.                      |
| Private instance cannot reach the internet | Check the private route table association and default route.                      |
| Public subnet cannot reach the internet    | Check the Internet Gateway attachment and public route table.                     |
| DNS resolution fails                       | Check VPC DNS settings and the instance's resolver configuration.                 |
| HTTPS requests fail                        | Check security group egress, network ACLs, DNS, and the external endpoint.        |
| Route creation fails                       | Verify the route table ID, destination, and target resource.                      |
| NAT Gateway charges continue               | Check whether the gateway remains provisioned and review data processing charges. |

## 14. Security Best Practices

* Keep private instances without public IP addresses unless specifically required.
* Restrict security group traffic to necessary ports and destinations.
* Use Systems Manager Session Manager where appropriate instead of exposing SSH.
* Remember that a NAT Gateway is not a firewall replacement.
* Review route tables and subnet associations carefully.
* Monitor AWS resource usage and costs.
* Remove lab resources when they are no longer needed.

## 15. Resource Cleanup

NAT Gateways and Elastic IP addresses can incur charges. Delete resources that are no longer needed, taking care not to remove resources shared with other labs.

First, delete the NAT Gateway:

```bash
aws ec2 delete-nat-gateway \
  --nat-gateway-id "$NAT_GW_ID" \
  --region "$AWS_REGION"
```

Wait until it has been deleted:

```bash
aws ec2 wait nat-gateway-deleted \
  --nat-gateway-ids "$NAT_GW_ID" \
  --region "$AWS_REGION"
```

Release the Elastic IP:

```bash
aws ec2 release-address \
  --allocation-id "$EIP_ALLOC_ID" \
  --region "$AWS_REGION"
```

After the NAT Gateway has been deleted, remove the default route from the new private route table if it remains:

```bash
aws ec2 delete-route \
  --route-table-id "$NEW_PRIVATE_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --region "$AWS_REGION"
```

Then disassociate and delete the additional private route table if it is no longer needed. Do not delete the existing VPC, original route table, private subnet, or shared Internet Gateway unless the lab explicitly requires it and you have confirmed that nothing else depends on them.

## 📚 Key Learnings

* Understanding public and private subnet routing.
* Configuring an Internet Gateway and NAT Gateway.
* Allocating and using an Elastic IP.
* Creating and associating route tables.
* Routing private subnet traffic through a NAT Gateway.
* Testing outbound connectivity.
* Understanding AWS networking security and cost management.

## 🏁 Conclusion

In Day 45, I configured a NAT Gateway to provide outbound internet access to an existing private VPC. I created an additional private route table, added a default route to the NAT Gateway, and associated the new route table with the private subnet.

This lab strengthened my understanding of AWS VPC networking, route tables, NAT Gateways, and private subnet internet connectivity.

**Day 45 of my 50 Days AWS Practice Lab Challenge — Zero to Hero!** ☁️🚀
