# Cross-Region-Failover-System 

# Building a Highly Available and Geographically Resilient Web Application on AWS

This project documents the end-to-end process of deploying a web application on AWS with a dual focus on **High Availability (HA)** to prevent local failures and **Disaster Recovery (DR)** to ensure business continuity during a regional outage.

The architecture is designed to achieve a low **Recovery Time Objective (RTO)**, minimizing downtime, and provides a framework for a low **Recovery Point Objective (RPO)**, minimizing data loss.

* **High Availability** is architected within a single region (`ap-south-1`) to withstand common failures, such as a server crash or a datacenter (Availability Zone) becoming unavailable.
* **Disaster Recovery** is achieved by replicating the infrastructure in a second, geographically separate region (`us-east-1`) and implementing an automated failover to redirect traffic if the entire primary region goes offline.

## 🏛️ Final Architecture

The final architecture is a robust, multi-region setup that routes users to a healthy application environment. Amazon Route 53 acts as the intelligent DNS layer, monitoring the primary site and triggering a failover to the DR site when necessary.

![Multi-Region HA/DR Architecture on AWS](https_placeholder_for_architecture_diagram.png)

**Traffic Flow:**
`User -> Route 53 (Failover Records with Health Check)`
* **Normal Operation:** `-> Primary HA Stack in ap-south-1 (Mumbai)`
* **After Regional Failure:** `-> Standby DR Stack in us-east-1 (N. Virginia)`

---

## 🚀 Deployment Walkthrough

The infrastructure was built by first establishing a highly available core in the primary region and then layering on a multi-region disaster recovery strategy.

### Part 1: Achieving High Availability (HA) in the Primary Region (`ap-south-1`)

The first step was to build a fault-tolerant stack to handle common issues without downtime.

1.  **Launch Template (`mytemp`):** A template was created to launch EC2 instances with a basic Apache web server. The user data script configures it to display: `<h1>Primary Zone A - Web Server</h1>`.
2.  **Multi-AZ Auto Scaling Group (`myauto`):** An Auto Scaling Group was configured to maintain a desired capacity of 3 instances. Its key HA features are:
    * **Self-Healing:** It automatically terminates and replaces any unhealthy EC2 instances.
    * **AZ-Redundancy:** It launches instances across multiple Availability Zones. If one AZ fails, the ASG continues to operate in the healthy zones.
3.  **Application Load Balancer (`mylb`):** The ALB distributes traffic to the instances and is also deployed across multiple Availability Zones, eliminating it as a single point of failure.

### Part 2: Implementing Multi-Region Disaster Recovery (DR)

To protect against an entire regional failure, a standby environment was created.

1.  **DR Site Replication (`us-east-1`):** A parallel infrastructure stack was deployed in a different geographical region, with its user data configured to display: `<h1>Disaster Recovery - Region B</h1>`.
2.  **Automated Failover with Route 53:**
    * **Health Check:** A Route 53 Health Check was created to constantly monitor the primary ALB's endpoint.
    * **Failover Routing Policy:** Two 'A' records were configured in the `awspractice.space` Hosted Zone—a **Primary** record pointing to the `mylb` load balancer and a **Secondary** record pointing to the `drload` load balancer, which becomes active only when the primary health check fails.

---

## ✅ Successful Failover Test

The system was validated by simulating a primary region failure. After the Health Check failed, Route 53 automatically redirected traffic to the DR site in `us-east-1`. The website remained accessible, serving the "Disaster Recovery - Region B" page, confirming a successful and seamless failover.

---

## 💡 In-Depth Architectural Considerations

### Understanding RTO and RPO

* **Recovery Time Objective (RTO):** The maximum acceptable time for an application to be offline after a disaster. This architecture achieves a **low RTO** (minutes) because the failover is automated by Route 53.
* **Recovery Point Objective (RPO):** The maximum acceptable amount of data loss, measured in time (e.g., 5 minutes of data). This architecture, as built, is for a **stateless application**. For a **stateful application**, achieving a low RPO requires a data replication strategy.

### Stateless vs. Stateful Applications

This project deploys a stateless application where each server is identical and holds no unique data. For a stateful application (e.g., with a database), you must also replicate your data across regions. Common strategies include:
* **Amazon Aurora Global Database:** Provides fast cross-region replication with sub-second latency, ideal for a low RPO.
* **RDS Cross-Region Read Replicas:** A cost-effective way to maintain a readable copy of your database in the DR region, which can be promoted to a primary database during failover.
* **DynamoDB Global Tables:** Provides a fully managed, multi-active, multi-region database.

### DR Cost Management Strategies

A full replica of your infrastructure can be expensive. You can manage DR costs with different models:
* **Hot Standby (this project):** A scaled-down but fully functional copy of the primary is always running in the DR region. RTO is very low.
* **Warm Standby:** A minimal version of the infrastructure is running (e.g., ASG with desired count = 1). RTO is longer as you need to scale up during a disaster.
* **Pilot Light:** Only the critical data is replicated. The infrastructure is defined in code (IaC) but not running until needed. This is the most cost-effective but has the longest RTO.

### Failback Planning

Failover is only half the process. A **failback** plan is needed to return traffic to the primary region once it is stable. This is typically a carefully managed manual process:
1.  Ensure the primary region infrastructure is fully restored and healthy.
2.  Replicate any data changes that occurred in the DR region back to the primary database.
3.  Update the Route 53 records to point back to the primary ALB, often done during a low-traffic maintenance window.

---

## 🔧 Potential Improvements & Next Steps

* **Infrastructure as Code (IaC):** Deploying this entire architecture using **AWS CloudFormation** or **Terraform** would make it repeatable, version-controlled, and less prone to human error.
* **Enhanced Security:** Integrate **AWS WAF** (Web Application Firewall) with the Application Load Balancers to protect against common web exploits. Tighten security group rules to only allow necessary traffic.
* **Automated Monitoring & Alerting:** Configure **Amazon CloudWatch Alarms** on the Route 53 health check status and other key metrics. Use **SNS (Simple Notification Service)** to send alerts to administrators during a failover event.
* **CI/CD Integration:** Build a CI/CD pipeline (e.g., using AWS CodePipeline) to automate application deployments, ensuring updates are pushed to both the primary and DR regions simultaneously for consistency.
