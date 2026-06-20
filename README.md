Question 1) Your EC2 has a public IP and the port is open in the security group, but it's unreachable. Why?

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
Stateful                                       
Applied to EC2 ENIs.                           
Allow rules only.                              
Return traffic automatically allowed.          

NACL
Stateless
Applied to Subnets
Allow and Deny rules
Return traffic must be explicitly allowed



Q2. Why do NACLs need both inbound and outbound rules?

Answer:

Because NACLs are stateless.
Client → EC2  (Inbound)
EC2 → Client  (Outbound Response)
If outbound is denied, the client never receives the response, even though the request reached the server.

⸻

Q3. How do you know if a subnet is public?

Answer:

A subnet is public if its Route Table contains:
Destination: 0.0.0.0/0
Target: Internet Gateway
Without that route, it’s effectively private.

Q4. The EC2 has a Public IP and an Internet Gateway. Why is it still unreachable?

Answer:

Possible reasons include:

* Network ACL blocking traffic.
* OS firewall (iptables, firewalld, ufw).
* Application not running.
* Application listening on the wrong port.
* Application bound only to 127.0.0.1 instead of 0.0.0.0.
* Route Table misconfiguration.

Q5. SSH is also not working. What will you check?

I would verify:

* EC2 instance state (Running)
* EC2 Status Checks (2/2 passed)
* Internet Gateway
* Route Table
* Security Group (port 22)
* Network ACL
* Public IP or Elastic IP
* OS firewall
* SSH service (sshd) is running

⸻

Q6.  Why do we need both a Security Group and a NACL?

Answer:

They protect at different layers:

* Security Group: Instance-level firewall with stateful filtering.
* Network ACL: Subnet-level firewall with stateless filtering.

Using both provides defense in depth.

⸻

Instead, demonstrate a structured troubleshooting process:

1. VPC – Internet Gateway attached.
2. Routing – Public Route Table.
3. Subnet – Public vs. Private.
4. NACL – Inbound and outbound rules.
5. Security Group – Correct ports and source.
6. EC2 – Instance health.
7. Operating System – Firewall and listening ports.
8. Application – Running and bound to the correct interface.
====================================================================
2) You shared an AMI with another AWS account, but they still can’t launch an instance from it. What’s usually missed?

Ans: Sharing the AMI isn’t enough. You also need to share the associated EBS snapshot. Without that, the AMI looks valid but fails at launch.

Sharing the AMI alone is not sufficient. An Amazon Machine Image (AMI) references one or more EBS snapshots that contain the actual disk data.

If I share only the AMI but don’t grant permission to the associated EBS snapshot(s), the other AWS account can see the AMI but won’t be able to launch an EC2 instance from it because it doesn’t have access to the underlying storage.

I also verify:

* The EBS snapshot is shared with the target AWS account.
* The AMI and snapshot are in the same AWS Region as expected.
* Any required KMS key permissions are granted if the snapshots are encrypted.
* The target account has permission to use the encrypted snapshot.

So my checklist is: AMI permissions, snapshot permissions, encryption/KMS permissions, and Region compatibility.

Real Production Example

We created a hardened Linux AMI and shared it with another AWS account for their production environment.

The other team could see the AMI in the console but received an error when launching an instance.

During troubleshooting, we found that only the AMI had been shared. The backing EBS snapshot permissions had not been updated.

After sharing the snapshot with the target account, the EC2 instance launched successfully.

Q1. What is an AMI?

Answer

An Amazon Machine Image (AMI) is a template used to launch EC2 instances.

It typically contains:

* Operating System
* Installed software
* Application packages
* Block device mapping
* Boot configuration

⸻

Q2. What is an EBS Snapshot?

Answer

An EBS snapshot is a point-in-time backup of an EBS volume stored in Amazon S3 (managed internally by AWS).

It is used for:

* Creating new EBS volumes
* Creating AMIs
* Disaster recovery
* Backup and restore

⸻

Q3. Why is the snapshot required?

Answer

The AMI contains metadata, but the snapshot contains the actual disk contents.

During launch:
AMI
   │
Reads Snapshot
   │
Creates EBS Volume
   │
Launches EC2

Without snapshot access, the volume cannot be created.

⸻

Q4. What if the snapshot is encrypted?

Answer

If the snapshot is encrypted with a customer-managed KMS key, the target account also needs permission to use that KMS key.

Sharing the snapshot alone is not enough.

⸻

Q5. Can you share an encrypted AMI?

Answer

Yes, but you must also share the customer-managed KMS key (if used) with the target account. If AWS-managed encryption keys are involved, there are additional sharing limitations.

⸻

Q6. Can you launch an AMI in another Region?

Answer

Not directly.

You first copy the AMI to the target Region. AWS copies the associated snapshots as part of the process.

⸻

Q7. Can one AMI have multiple snapshots?

Answer

Yes.

If the instance has multiple EBS volumes, the AMI can reference multiple snapshots—one for each volume.

Example:

Q8. How do you share an AMI?

Answer

Typical process:

1. Create the AMI.
2. Share the AMI with the target AWS account.
3. Share the associated EBS snapshot(s).
4. If encrypted, grant access to the KMS key.
5. The target account launches the EC2 instance.

⸻

Q9. Can you modify an AMI after creating it?

Answer

No.

AMIs are immutable. If changes are required, launch an EC2 instance, make the changes, and create a new AMI.

⸻

Q10. Why do organizations use custom AMIs?

Answer

Custom AMIs help standardize deployments by including:

* Security patches
* Monitoring agents
* CloudWatch Agent
* Company-approved software
* Application runtime
* Security hardening

This reduces provisioning time and ensures consistency.

                Create AMI
                    │
                    ▼
         Amazon Machine Image (AMI)
                    │
                    ▼
      References EBS Snapshot(s)
                    │
                    ▼
        Share AMI with Account B
                    │
                    ▼
     Share EBS Snapshot(s) Too
                    │
                    ▼
       (If encrypted) Share KMS Key
                    │
                    ▼
      Account B Launches EC2 Instance

“The first thing I’d verify is whether the backing EBS snapshot has been shared. If the snapshot is encrypted, I’d also check KMS key permissions. An AMI without access to its underlying snapshots cannot be used to launch an EC2 instance.”









