# Secure AWS Environment
This project demonstrates how to design and deploy a secure AWS environment in line with security best practices. 
The focus is on access control, network isolation, logging, and cost management.

## Project Scope 
This project focuses on securing a basic AWS environment and does not include application development, CI/CD Pipelines, or production workloads. 

## Threat Model 
By default, AWS accounts are vulnerable to misconfiguration rather than platform-level flaws.
This project focuses on reducing the most common risks in small AWS environments 

### Key Risks Addressed 
- **Excessive IAM Permissions** - Overly broad permissions can lead to accidental or malicious changes.
- **Public Network Exposure** - Resources exposed directly to the internet can increase the attack surface.
- **publicly Accessible Storage** - Misconfigured S3 buckets can lead to data exposure.
- **Lack of Visibility** - Without logging and monitoring, security incidents can go undetected.
- **Uncontrolled Costs** - Forgotten resources can generate unexpected charges. 
