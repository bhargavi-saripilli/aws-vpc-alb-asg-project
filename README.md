\## AWS Production-Style VPC: ALB + Auto Scaling in Private Subnets



A highly available web application on AWS. The servers run in "private subnets across two Availability Zones" and receive traffic only through an "Application Load Balancer". They reach the internet through "NAT gateways".



> Based on Abhishek Veeramalla's AWS series. I rebuilt it on my own AWS account and debugged the issues listed below.



\#Architecture



| Component | Details |

|---|---|

| VPC | 10.0.0.0/16, 2 public + 2 private subnets in us-east-1a and us-east-1b |

| Internet Gateway | Routes public subnets to the internet |

| NAT Gateways | One per AZ, so private servers can download packages |

| Application Load Balancer | Internet-facing, listener HTTP:80 forwards to the target group |

| Target Group | HTTP port 8000, health checked |

| Auto Scaling Group | Launch template, min 1, desired 2, max 4, replaces unhealthy instances |

| Security Group | HTTP 80 open; port 8000 allowed only from within the same security group |



\#Startup script (launch template user data)



```bash

\#!/bin/bash

mkdir -p /app

echo "<h1>Hello from $(hostname -f)</h1>" > /app/index.html

cd /app

setsid nohup python3 -m http.server 8000 > /var/log/app.log 2>\&1 \&

```



\# Result



Opening the load balancer DNS name returns a page from a server in a private subnet.







!\[Browser output](images/1-browser-hello.png)







Both targets healthy in the target group:







!\[Healthy targets](images/2-targets-healthy.png)







Load balancer resource map:







!\[Resource map](images/3-resource-map.png)







Auto Scaling group with two instances in two Availability Zones:







!\[Auto Scaling instances](images/4-asg-instances.png)







\# Problems I troubleshot



1\. "One target stayed unhealthy (us-east-1b)."

&#x20;  I compared the security groups, route tables, NAT gateway and network ACL, which were all correct. The launch template had "no user data", so a replacement instance never started a web server on port 8000. Adding a startup script to the launch template fixed it for every new instance.

2\. "Replacement instance did not launch at first."

&#x20;  The Auto Scaling activity history showed a "vCPU limit" error. The group retried and succeeded once capacity was free. Stopping unused instances frees capacity.

3\. "Browser showed "This site can't be reached"."

&#x20;  The browser switched to `https://`, but the listener is HTTP port 80 only. I confirmed the app with `curl` (200 OK) and opened the address with `http://`.



\# What I learned



\- Why servers go in private subnets and how a NAT gateway gives them outbound access

\- How health checks, target groups and Auto Scaling replace failed servers

\- Reading Auto Scaling activity history and EC2 system logs to find the cause of a failure

\- Why startup configuration belongs in the launch template, not on one hand-configured server



\# Clean-up



NAT gateways and the load balancer bill by the hour, so I deleted them after testing: Auto Scaling group set to 0, load balancer deleted, NAT gateways deleted, Elastic IPs released.

