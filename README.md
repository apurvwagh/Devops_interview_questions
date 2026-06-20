1) Your EC2 has a public IP and the port is open in the security group, but it's unreachable. Why?

Ans: Check the subnet’s NACL. If inbound or outbound rules are blocking traffic, the security group won’t help. NACLs silently drop traffic with no message.
in Details 

If an EC2 instance has a public IP and the required port is allowed in the Security Group, but it’s still unreachable, I troubleshoot layer by layer instead of assuming the Security Group is the problem.

My checklist is:

1. Verify the Internet Gateway (IGW)

* Ensure the VPC has an Internet Gateway attached.

2. Check the Route Table

* Confirm the subnet’s route table has a default route (0.0.0.0/0) pointing to the Internet Gateway.

3. Verify the subnet is Public

* A public IP alone doesn’t make a subnet public. The subnet must have a route to the Internet Gateway.

4. Check the Network ACL (NACL)

* Since NACLs are stateless, both inbound and outbound rules must allow the traffic.
* If inbound allows the request but outbound blocks the response (or vice versa), the connection fails.

5. Verify the Security Group

* Ensure the required port (e.g., 22, 80, or 443) is allowed from the correct source.

6. Check the EC2 Instance

* Verify the application is running.
* Confirm it’s listening on the expected port using ss -tulnp or netstat.

7. Check the Operating System Firewall

* Verify iptables, firewalld, or ufw isn’t blocking the port.

8. Verify the Application

* Ensure the application is bound to the correct interface (0.0.0.0) and not just 127.0.0.1.

My goal is to identify exactly where the request is being blocked instead of assuming it’s an AWS networking issue.

Real Production Scenario

We had an EC2 instance running Nginx with a public IP. The Security Group allowed port 443, but users couldn’t access the application.

I verified the Security Group first, which was correct. Then I checked the subnet’s Network ACL and found that outbound ephemeral ports (1024–65535) were denied. Since NACLs are stateless, the response packets were blocked.

After updating the outbound NACL rule, connectivity was restored.

⸻

Cross Questions from the Interviewer

Q1. What is the difference between Security Group and NACL?

Security Group

NACL

Stateful

Stateless

Applied to EC2 ENIs

Applied to Subnets

Allow rules only

Allow and Deny rules

Return traffic automatically allowed

Return traffic must be explicitly allowed









=============================================================================================================================================================================
