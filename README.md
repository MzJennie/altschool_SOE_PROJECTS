🚀 Deploying a Secure Web Application on AWS Using Bastion, Ansible & Load Balancer

As part of my journey into Cloud Architecture and DevOps, I designed and deployed a secure, production-style web application architecture on AWS, and this article walks through the architecture, settings, and challenges encountered.

🏗 Architecture Overview

The infrastructure was deployed in a custom VPC with the following components:
- 1 Bastion Host (Public Subnet)
- 2 Web Servers (Private Subnet)
- 1 Application Load Balancer (Public Subnets)
- NAT Gateway (for outbound internet from private subnet)
- Ansible for automated configuration

🖼 Architecture Diagram

                     Internet
                         │
                 ┌───────────────┐
                 │ Application   │
                 │ Load Balancer │
                 └───────┬───────┘
                         │
        ┌────────────────┴────────────────┐
  
        │                                 │
 ┌───────────────┐                 ┌───────────────┐

 │ Web Server 1  │                 │ Web Server 2  │
 
 │ (Private Sub) │                 │ (Private Sub) │
 
 └───────────────┘                 └───────────────┘
        │
        
        │ (Outbound only)
        ▼
   ┌───────────────┐
   
   │  NAT Gateway  │
   
   └───────────────┘
        
        │
        
   ┌───────────────┐
 
   │ Internet GW   │
   
   └───────────────┘
   
        ▲
        │
  
   ┌───────────────┐
   
   │ Bastion Host  │
   
   │ (Public Sub)  │
   
   └───────────────┘
   
        ▲
        │ SSH Access
     My Local Machine

🔐 Why I Used a Bastion Host

A Bastion Host (Jump Server) is a security best practice, and it helps limit direct internet exposure of application servers. Instead of giving public access to every server:
- Only the Bastion has a public IP
- Web servers have no public IP
- SSH access to web servers is allowed only from the Bastion Security Group

⚙ How Ansible Simplified Deployment

Rather than manually installing NGINX on each server, I used Ansible automation with the command below;

ansible-playbook -i inventory.ini deploy-nginx.yml

So instead of logging into 2 servers and configuring manually, the playbook performed these automatically to both servers:

- NGINX installation
- Service startup and enablement
- Deployment of a dynamic HTML page
- Hostname-based instance identification

This helps ensure consistency across environments, and repeatable deployments.

🌍 Direct EC2 Access vs Load Balancer Access

❌ Direct EC2 Access - If the web servers had public IPs, this means that anyone can access them directly

User Browser
    
    ↓
    
    [Internet]
    
    ↓

EC2 Instance (Public IP)

    ↓

NGINX on Port 80

    ↓

 Web Application


✅ Load Balancer Access

With an Application Load Balancer: Only ALB is internet-facing, and the web servers remain private.
Also, traffic is distributed evenly, and refreshing the ALB DNS showed alternating instance hostnames — confirming load balancing worked.

I encountered some challenges during the task as seen below;

⚠ Challenge 1: SSH From Bastion to Private Web Servers Failed

While testing secure access, I successfully SSH’d into the Bastion:
ssh -A -i ~/key_pairs.pem ubuntu@<BASTION_PUBLIC_IP>

However, attempting to SSH from Bastion to the private web server (ssh ubuntu@10.0.2.196) resulted in a Permission denied (publickey) error.

Running: ssh-add -L
Returned: Could not open a connection to your authentication agent.

During troubleshooting, I noticed that although I used -A (agent forwarding), my local SSH agent was not running. Also, no private key was loaded into the authentication agent. 
Hence, Bastion could not access my key, which was why authentication to private servers failed.

✅ Solution - 
I ran the following 3 commands on my Vagrant VM:

1️⃣ - eval "$(ssh-agent -s)"

This started the SSH authentication agent, Set the required environment variables, and enabled the shell to communicate with the agent

2️⃣ - ssh-add ~/key_pairs.pem

This added my private key to the agent, and made it available for authentication and forwarding

3️⃣ - ssh-add -l

This listed the loaded keys, and confirmed successful configuration

After reconnecting with my Bastion public IP -
ssh -A -i ~/key_pairs.pem ubuntu@108.131.138.159

I successfully accessed both private web servers!


⚠ Challenge 2: Ansible Playbook Failed (No Internet in Private Subnet)

The Ansible playbook failed during Install NGINX. This error occurred because:

- Web servers were inside a private subnet
- No outbound internet route existed
- No NAT Gateway configured
- apt update could not reach Ubuntu repositories
- Private subnets do not have internet access by default.

✅ Solution: Configure NAT Gateway

To enable outbound internet:

1️⃣ Allocated Elastic IP

EC2 → Elastic IPs → Allocate Elastic IP

2️⃣ Created NAT Gateway

- VPC → NAT Gateways → Create NAT Gateway
- Connectivity type: Public
- Subnet: Public subnet
- Attached Elastic IP

3️⃣ Update the Private Route Table

- VPC → Route Tables → private-rt → Edit routes
- Destination: 0.0.0.0/0
- Target: NAT Gateway

After this, the private web servers gained outbound internet access and apt update completed successfully, and the Ansible playbook ran successfully as seen below - 

ansible-playbook -i inventory.ini deploy-nginx.yml

PLAY [Configure Web Servers] *******************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [webserver2]
ok: [webserver1]

TASK [Show target host] ************************************************************************************************
ok: [webserver1] => {
    "msg": "Deploying to webserver1"
}
ok: [webserver2] => {
    "msg": "Deploying to webserver2"
}

TASK [Install NGINX] ***************************************************************************************************
changed: [webserver1]
changed: [webserver2]

TASK [Ensure NGINX is running] *****************************************************************************************
ok: [webserver2]
ok: [webserver1]

TASK [Deploy HTML page] ************************************************************************************************
changed: [webserver1]
changed: [webserver2]

PLAY RECAP *************************************************************************************************************
webserver1                 : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
webserver2                 : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0



