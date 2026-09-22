# Project Walkthrough: Securing AWS Environments with IAM & ABAC

## 🎯 Project Overview
- **Objective:** Securely onboard a new team member (an intern) by granting them access *only* to development resources, while completely protecting production infrastructure from accidental or malicious changes.
- **Cloud Provider:** AWS
- **Core Services Used:** Amazon EC2, AWS IAM, IAM Policy Simulator
- **Security Domain:** Identity & Access Management (IAM)

##  Security Concepts & Architecture
Before building, I designed this environment around three core cloud security principles:
1. **Attribute-Based Access Control (ABAC):** Instead of hardcoding instance IDs into security policies, I used metadata tags (e.g., `Env: development`) to dynamically control access.
2. **Role-Based Access Control (RBAC):** Permissions are assigned to IAM Groups rather than directly to individual users, making the architecture highly scalable.
3. **Principle of Least Privilege (PoLP):** The intern is given the exact permissions needed to do their job, and explicitly denied from doing anything else.

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/6f70aedb-cd41-4d8f-a5db-32f80cf01a5f" />


---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Provisioning & Tagging EC2 Instances
To simulate the company's infrastructure, I navigated to the EC2 console and launched two virtual servers using the Free Tier Amazon Machine Image (AMI) and `t3.micro` instance type. 

The most critical part of this step was applying **Tags**. Tags act as the foundation for our ABAC security policy.
* I deployed the first instance as `network-prod-vishwas` and tagged it with **Key:** `Env`, **Value:** `production`.
* I deployed the second instance as `network-dev-vishwas` and tagged it with **Key:** `Env`, **Value:** `development`.

<img width="1599" height="321" alt="image" src="https://github.com/user-attachments/assets/76a048f9-6538-4f0c-b424-d3b92b4fec99" />


By checking the tags tab on the instances, I verified that the metadata was successfully attached. 

<img width="865" height="277" alt="image" src="https://github.com/user-attachments/assets/e4c71c7b-2a6b-4e68-bfb7-2bcafbc8005c" />


### Step 2: Authoring a Custom IAM Policy (JSON)
Next, I needed to create the permission rules for the intern. I navigated to **IAM > Policies** and created a custom policy named `DevEnvironmentPolicy` using the following JSON script:

```json
{    
  "Version": "2012-10-17",    
  "Statement": [        
    {            
      "Effect": "Allow",            
      "Action": "ec2:*",            
      "Resource": "*",            
      "Condition": {                
        "StringEquals": {                    
          "ec2:ResourceTag/Env": "development"                
        }            
      }        
    },        
    {            
      "Effect": "Allow",            
      "Action": "ec2:Describe*",            
      "Resource": "*"        
    },        
    {            
      "Effect": "Deny",            
      "Action": [                
        "ec2:DeleteTags",                
        "ec2:CreateTags"            
      ],            
      "Resource": "*"        
    }    
  ] 
}
```

**Breaking down the Security Logic:**
* **The Allow Block:** Grants full EC2 control (`ec2:*`), but *strictly enforces a condition*—the resource must possess the `Env: development` tag.
* **The Describe Block:** Grants read-only visibility (`Describe*`) across all EC2 instances so the user can actually view the dashboard.
* **The Deny Block (Privilege Escalation Prevention):** Explicitly blocks the creation or deletion of tags. Without this, the intern could simply rename the production server's tag to "development" and bypass our entire security system!

<img width="1529" height="308" alt="image" src="https://github.com/user-attachments/assets/50a14999-807a-4f83-994f-747dd1fc047e" />


### Step 3: Streamlining Login with an Account Alias
By default, the AWS console login URL requires a hard-to-remember 12-digit Account ID. To obfuscate the root account ID and make onboarding smoother, I created an Account Alias: `org-alias-vishwas`. 

### Step 4: Setting up IAM Groups and the User
Following enterprise best practices, I avoided attaching the policy directly to the intern. 

1. **Creating the Group:** I created an IAM User Group called `Dev-group` and attached my custom `DevEnvironmentPolicy` to it.

<img width="1562" height="281" alt="image" src="https://github.com/user-attachments/assets/c9e8c958-548f-4f1c-b0d8-d849ffb8b503" />


2. **Creating the User:** I created a new IAM User named `Intern_Adithya`, enabled Management Console access, generated a password, and added them to the `Dev-group`.

<img width="1516" height="319" alt="image" src="https://github.com/user-attachments/assets/e3716bf5-4ef3-4ee2-b4e0-9ab988964ce8" />


The user now has a clean login portal customized with our account alias.

<img width="300" height="400" alt="image" src="https://github.com/user-attachments/assets/50a7672f-b0f7-47ba-adac-5e52c1b81b35" />


---

## 🔬 Verification & Testing (Proving the Controls)
To ensure the security policies functioned correctly in a real-world scenario, I opened a private/incognito browser window and logged into AWS as `Intern_Adithya`. 

Immediately, the restricted permissions took effect. Navigating to the account dropdown menu showed "Access denied" for administrative account-level configurations.

<img width="300" height="300" alt="image" src="https://github.com/user-attachments/assets/84c7128f-e741-46d3-b9ac-b766cec0f02a" />


**Test 1: Attempting to Compromise the Production Server (Expected: Blocked)**
I navigated to the EC2 console, selected the `network-prod-vishwas` instance, and clicked "Stop Instance". 
Because this instance is tagged as `production`, the IAM policy successfully blocked the action, throwing a verbose authorization failure error. The production environment is successfully isolated.

<img width="1581" height="586" alt="image" src="https://github.com/user-attachments/assets/f0df66eb-367f-44b2-93e8-8a6ce82fb711" />


**Test 2: Working in the Development Server (Expected: Allowed)**
Next, I selected the `network-dev-vishwas` instance and attempted to stop it. 
Because this instance possesses the exact `Env: development` tag defined in our policy condition, the action was allowed, and the instance successfully began to stop.

<img width="1574" height="533" alt="image" src="https://github.com/user-attachments/assets/21847c4d-fecb-4557-bbc9-7d799adab5a3" />


---

##  Advanced Testing: IAM Policy Simulator
Instead of manually clicking through the console to test every edge case, I utilized the **AWS IAM Policy Simulator**—a powerful tool for Security Engineers to mathematically validate access controls.

I configured a simulation for the `Intern_Adithya` identity, specifically testing the `StopInstances` and `DeleteTags` actions.

<img width="1878" height="678" alt="image" src="https://github.com/user-attachments/assets/64c93432-a118-44c6-b680-101ee83d7dad" />


The simulator confirmed our architecture is secure without needing to touch live infrastructure:
* Attempting to stop instances without the proper tags resulted in an **Implicit Deny**.
* Attempting to manipulate tags resulted in an **Explicit Deny** (triggered directly by the Deny block in our JSON policy).

<img width="1341" height="443" alt="image" src="https://github.com/user-attachments/assets/c3258360-7c19-49e5-896d-01dda75251bd" />

---

## 💡 Security Best Practices Applied
- [x] **Separation of Environments:** Used EC2 instance metadata (Tags) to physically and logically separate Prod and Dev.
- [x] **Group-Level Access:** Scaled permission management by utilizing IAM Groups rather than direct user attachment.
- [x] **Explicit Deny Statements:** Weaponized the `Deny` effect to enforce immutability on resource tags, preventing ABAC bypass attacks.

## 📖 Lessons Learned / Analyst Reflection
This project practically demonstrated the power of combining standard **Role-Based Access Control (RBAC)** with **Attribute-Based Access Control (ABAC)**. I learned that simply granting access to a group is often not granular enough for enterprise environments. Writing custom JSON policies gave me a deeper appreciation for how condition keys and explicit Deny statements are absolutely critical for closing privilege escalation loopholes.
