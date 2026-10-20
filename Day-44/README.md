# Day 44: Implementing Auto Scaling for High Availability in AWS

## 📌 Project Overview

Welcome to Day 44 of my **50 Days AWS Practice Lab Challenge — Zero to Hero!**

In this hands-on lab, I explored how to deploy a highly available web application on AWS using an **EC2 Auto Scaling Group, an Application Load Balancer, and Nginx**.

The Application Load Balancer distributes incoming HTTP traffic across healthy EC2 instances. The Auto Scaling Group maintains the desired number of instances and can launch replacement instances when existing instances become unhealthy.

This lab demonstrates how AWS helps improve application availability, scalability, and fault tolerance.

## 🎯 Objectives

* Understand EC2 Auto Scaling Groups.
* Create and configure an EC2 launch template.
* Deploy Nginx on EC2 instances.
* Configure an Application Load Balancer.
* Create a target group and health checks.
* Deploy instances across multiple Availability Zones.
* Attach the Auto Scaling Group to the load balancer.
* Test traffic distribution and instance replacement.
* Understand AWS resource cleanup and cost management.

## 🛠️ AWS Services Used

* **Amazon EC2:** Runs the Nginx web servers.
* **EC2 Auto Scaling:** Maintains and adjusts the number of EC2 instances.
* **Application Load Balancer (ALB):** Distributes incoming HTTP requests.
* **Amazon VPC:** Provides the network environment.
* **EC2 Launch Template:** Defines the configuration for launched instances.
* **Target Group:** Registers instances and checks their health.
* **Amazon CloudWatch:** Provides monitoring and metrics for scaling.

## 🏗️ Architecture

```text
                       Users / Browser
                              |
                              v
                  Application Load Balancer
                              |
                              v
                        Target Group
                         /         \
                        v           v
                  EC2 Instance 1  EC2 Instance 2
                     Nginx          Nginx
                        \           /
                         Auto Scaling
                             Group
                    (Multiple Availability Zones)
```

### How It Works

1. A user sends an HTTP request to the ALB DNS endpoint.
2. The ALB forwards the request to a healthy registered EC2 instance.
3. Nginx processes the request and returns a response.
4. The Auto Scaling Group maintains the configured instance capacity.
5. When an instance becomes unhealthy, Auto Scaling can replace it.
6. When a configured scaling policy is triggered, Auto Scaling can increase or decrease capacity within the configured limits.

## 📋 Prerequisites

* An AWS account with the required permissions.
* Access to the AWS Management Console.
* An existing VPC with suitable subnets in at least two Availability Zones.
* A key pair if SSH access is required.
* An appropriate EC2 instance type and Amazon Linux AMI.
* An understanding of basic EC2 and VPC networking.

## 1. Create a Security Group for the Load Balancer

1. Open the **EC2 console**.
2. Select **Security Groups**.
3. Choose **Create security group**.
4. Enter a name such as `Day44-ALB-SG`.
5. Select the VPC used for the lab.
6. Configure the inbound rule:

| Type | Protocol | Port | Source      |
| ---- | -------- | ---- | ----------- |
| HTTP | TCP      | 80   | `0.0.0.0/0` |

7. Configure outbound rules according to your requirements.
8. Create the security group.

For a public demonstration website, HTTP access from the internet allows browsers to reach the ALB. For production, prefer HTTPS with an appropriate TLS certificate.

## 2. Create a Security Group for EC2 Instances

1. Open **EC2 → Security Groups**.
2. Choose **Create security group**.
3. Enter a name such as `Day44-WebServer-SG`.
4. Select the same VPC.
5. Add the following inbound rule:

| Type | Protocol | Port | Source                        |
| ---- | -------- | ---- | ----------------------------- |
| HTTP | TCP      | 80   | `Day44-ALB-SG` security group |

6. Add SSH access only if required, restricted to a trusted source.
7. Create the security group.

**Security best practice:** Allow HTTP traffic from the ALB security group instead of allowing direct HTTP access from the entire internet to every EC2 instance.

## 3. Create an EC2 Launch Template

The launch template defines how Auto Scaling creates new EC2 instances.

1. Open **EC2 → Launch Templates**.
2. Choose **Create launch template**.
3. Enter a name such as `Day44-Nginx-Launch-Template`.
4. Select an Amazon Linux AMI.
5. Choose an appropriate instance type.
6. Select or configure a key pair if needed.
7. Under network settings, select `Day44-WebServer-SG`.
8. Configure storage as required.
9. Under **Advanced details → User data**, provide a startup script to install and start Nginx.

Example user data for Amazon Linux 2023:

```bash
#!/bin/bash
dnf install -y nginx
systemctl enable nginx
systemctl start nginx

INSTANCE_ID=$(curl -s \
  http://169.254.169.254/latest/meta-data/instance-id)

cat > /usr/share/nginx/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
  <title>AWS Auto Scaling Lab</title>
</head>
<body>
  <h1>Day 44 - AWS Auto Scaling Lab</h1>
  <p>Nginx web server is running successfully.</p>
  <p>Instance ID: ${INSTANCE_ID}</p>
</body>
</html>
EOF

systemctl restart nginx
```

10. Review the configuration.
11. Choose **Create launch template**.

The instance ID displayed on the page helps identify which EC2 instance handled a request.

**Note:** User data normally runs during the instance's initial launch. If the instance cannot download Nginx, check its outbound network access, package repositories, and security configuration.

## 4. Create a Target Group

The target group defines where the ALB sends requests and how it checks instance health.

1. Open **EC2 → Target Groups**.
2. Choose **Create target group**.
3. Select **Instances** as the target type.
4. Enter a name such as `Day44-Nginx-Target-Group`.
5. Select **HTTP** as the protocol and port `80`.
6. Select the VPC used for the lab.
7. Configure the health check:

   * Protocol: HTTP
   * Path: `/`
8. Create the target group.

The Auto Scaling Group will register its EC2 instances with this target group.

## 5. Create the Application Load Balancer

1. Open **EC2 → Load Balancers**.
2. Choose **Create load balancer**.
3. Select **Application Load Balancer**.
4. Enter a name such as `Day44-Application-Load-Balancer`.
5. Choose **Internet-facing** if the application should be publicly accessible.
6. Select the VPC used for the lab.
7. Select subnets in at least two Availability Zones.
8. Choose the `Day44-ALB-SG` security group.
9. Configure an HTTP listener on port `80`.
10. Set its default forwarding action to `Day44-Nginx-Target-Group`.
11. Review and create the load balancer.

Wait until the load balancer state becomes **Active**.

## 6. Create the Auto Scaling Group

1. Open **EC2 → Auto Scaling Groups**.
2. Choose **Create Auto Scaling group**.
3. Enter a name such as `Day44-Nginx-ASG`.
4. Select `Day44-Nginx-Launch-Template`.
5. Choose the appropriate launch template version.
6. Select the VPC used for the lab.
7. Select subnets in at least two Availability Zones.
8. Configure the desired capacity settings.

Example settings:

| Setting          | Example value |
| ---------------- | ------------: |
| Minimum capacity |             2 |
| Desired capacity |             2 |
| Maximum capacity |             4 |

These are example values; use the values from your actual lab.

9. Configure load balancing and attach the existing `Day44-Nginx-Target-Group`.
10. If available, enable Elastic Load Balancing health checks for the Auto Scaling Group.
11. Configure additional monitoring or scaling policies if required by the lab.
12. Review the configuration and create the Auto Scaling Group.

The group should launch instances according to the desired capacity, subject to available capacity, subnet configuration, and service limits.

## 7. Verify the Auto Scaling Group

1. Open **EC2 → Auto Scaling Groups**.
2. Select `Day44-Nginx-ASG`.
3. Review the details and instance management tabs.
4. Confirm that the group is attempting to maintain the configured desired capacity.
5. Open **EC2 → Instances**.
6. Verify that the instances launched successfully and are running.

If the instances fail to launch, review the Auto Scaling activity history for the reason.

## 8. Verify Target Group Health

1. Open **EC2 → Target Groups**.
2. Select `Day44-Nginx-Target-Group`.
3. Open the **Targets** tab.
4. Check the health status of the registered instances.

Wait for the instances to become **Healthy**.

If an instance remains unhealthy, check:

* Whether Nginx is installed and running.
* Whether the health-check path returns a successful response.
* Whether the instance security group allows HTTP from the ALB security group.
* Whether network ACL rules permit the required traffic.
* Whether the target group is configured for the correct port and protocol.

## 9. Test the Application Using the ALB DNS Name

1. Open **EC2 → Load Balancers**.
2. Select `Day44-Application-Load-Balancer`.
3. Copy its **DNS name**.
4. Open a web browser.
5. Visit the load balancer's DNS name using HTTP if that is the listener configured for the lab.

Example:

```text
http://<YOUR-ALB-DNS-NAME>
```

Replace the placeholder with the DNS name displayed in your AWS console.

The page should display the Nginx test page created by the user data script.

Refresh the page and check the instance ID. Depending on the load-balancing behavior and the number of healthy targets, requests may reach different instances. Refreshing alone does not guarantee that every instance will be shown.

## 10. Test Auto Scaling and Instance Replacement

To understand how Auto Scaling maintains capacity:

1. Open the Auto Scaling Group.
2. Check the desired capacity and current instance count.
3. In a controlled lab, terminate one of the instances managed by the group.
4. Review the Auto Scaling Group's activity history.
5. Wait for the group to launch a replacement instance.
6. Verify that the new instance becomes healthy in the target group.
7. Visit the ALB DNS name again.

The Auto Scaling Group should attempt to restore its desired capacity. Replacement and health-check timing can vary.

**Important:** Only terminate an instance created for this lab. Do not terminate unrelated or production instances.

## 11. Configure a Scaling Policy (Optional)

A scaling policy can increase or decrease the number of instances in response to demand.

For example, a target-tracking policy can use average CPU utilization.

1. Open **EC2 → Auto Scaling Groups**.
2. Select `Day44-Nginx-ASG`.
3. Open the **Automatic scaling** tab.
4. Choose the option to create a dynamic scaling policy.
5. Select **Target tracking scaling**.
6. Select an appropriate metric, such as average CPU utilization.
7. Set a target value appropriate for your test.
8. Review and create the policy.

Auto Scaling adjusts capacity according to the selected metric and policy, within the minimum and maximum capacity limits.

CPU-based scaling might not trigger if the workload does not create enough CPU usage. Do not claim that scaling was tested unless you observed the policy change capacity.

## 12. Optional AWS CLI Verification

The implementation described in this lab is primarily console-based. The following commands can be used for verification if AWS CLI is already configured.

List Auto Scaling Groups:

```bash
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize}' \
  --output table
```

Check instances managed by the Auto Scaling Group:

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names Day44-Nginx-ASG \
  --query 'AutoScalingGroups[].Instances[].{InstanceId:InstanceId,Health:HealthStatus,Lifecycle:LifecycleState}' \
  --output table
```

Check the target group's registered targets:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --output table
```

Replace `<TARGET_GROUP_ARN>` with the actual target group ARN before running the command.

These commands are optional verification steps, not a requirement to perform the lab.

## 13. Troubleshooting

| Issue                                       | Possible solution                                                                                               |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| EC2 instances fail to launch                | Review the Auto Scaling activity history, launch template, subnet configuration, quotas, and instance capacity. |
| Nginx page does not load                    | Check the user data script, Nginx service, and instance logs.                                                   |
| Target instances are unhealthy              | Verify the health-check path, target port, security groups, and Nginx status.                                   |
| ALB DNS name does not respond               | Check the ALB state, listener, target group, and ALB security group.                                            |
| Only one instance appears to serve requests | Check the healthy targets and test behavior; repeated requests are not guaranteed to alternate instances.       |
| Auto Scaling does not add instances         | Check scaling policies, CloudWatch metrics, cooldown or warm-up behavior, and maximum capacity.                 |
| Instances repeatedly fail health checks     | Review application startup time, health-check settings, and networking.                                         |

## 14. Security Best Practices

* Allow public HTTP/HTTPS traffic through the load balancer rather than exposing every web server directly.
* Restrict instance security group inbound traffic to the load balancer security group.
* Use HTTPS with a valid TLS certificate for production applications.
* Use an IAM instance profile when instances need AWS API access.
* Avoid storing credentials in user data.
* Monitor Auto Scaling activity and application health.
* Use multiple Availability Zones for improved resilience.
* Review AWS resource costs after testing.

## 15. Cleanup

Resources created for this lab can incur charges. After completing your tests:

1. Delete the Auto Scaling Group when it is no longer needed. If prompted, choose the appropriate option to terminate the instances created by the group.
2. Wait for the instances to terminate.
3. Delete the Application Load Balancer.
4. Delete the target group after it is no longer associated with the load balancer or Auto Scaling Group.
5. Delete the launch template if it is no longer needed.
6. Remove lab-specific security groups only after confirming that no other resources depend on them.

Do not delete shared VPCs, subnets, or security groups used by other workloads.

## 📚 Key Learnings

* Understanding Amazon EC2 Auto Scaling Groups.
* Creating launch templates for repeatable instance configuration.
* Deploying Nginx using EC2 user data.
* Configuring an Application Load Balancer and target group.
* Using health checks to identify unhealthy targets.
* Maintaining instance capacity through Auto Scaling.
* Understanding scaling policies and capacity limits.
* Improving application availability across multiple Availability Zones.
* Troubleshooting and cleaning up AWS resources.

## 🏁 Conclusion

In Day 44 of my 50 Days AWS Practice Lab Challenge, I explored how to build a scalable web application using an EC2 Auto Scaling Group, an Application Load Balancer, and Nginx.

The lab demonstrated how a load balancer routes requests to healthy instances while Auto Scaling helps maintain application capacity and replace unhealthy instances.

This exercise improved my understanding of AWS high availability, load balancing, automated instance management, and cloud infrastructure troubleshooting.

**Day 44 completed — learning AWS one lab at a time! ☁️🚀**

---

*Note: Update the example resource names, capacity values, and testing results to match the configuration shown in the actual video and your AWS environment.*
