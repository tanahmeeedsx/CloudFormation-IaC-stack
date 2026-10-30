# bongoDev Student Lab - Infrastructure as Code with AWS CloudFormation

**Project:** Deploy Ubuntu 24.04 LTS + Nginx using one CloudFormation YAML template, then reuse the **same file** to create a separate Test environment.

**Level:** Beginner  |  **Estimated hands-on time:** 25–40 minutes (AWS provisioning time varies)  |  **Tools:** AWS Management Console and a web browser

> **Important:** This lab creates real, billable AWS resources. Complete the cleanup section when you finish, whether the lab succeeds or fails.

## 1. Learning objectives

By the end of the lab, you should be able to:

1. Describe Infrastructure as Code (IaC) in simple terms.
2. Deploy infrastructure by uploading a YAML file to **AWS CloudFormation**.
3. Find the VPC, subnet, security group, and EC2 instance created automatically.
4. Open a live **Nginx website running on Ubuntu 24.04 LTS**.
5. Create **Development** and **Testing** environments from the **same** template.
6. Delete all resources using CloudFormation.

## 2. What are we building?

```text
           cloudformation.yaml (same file)
                  /          \
                 /            \
       Dev CloudFormation    Test CloudFormation
             stack                stack
               |                    |
         Own VPC/subnet        Own VPC/subnet
         Internet gateway      Internet gateway
         Security group        Security group
         Ubuntu EC2            Ubuntu EC2
         Nginx website         Nginx website
       Environment: dev      Environment: test
```

Each stack creates its own AWS resources. **We are not copying an EC2 machine or sharing one VPC** - we are creating two independent environments using one infrastructure definition.

### What the YAML creates per stack

| Resource | Purpose |
|---|---|
| VPC (`10.42.0.0/16`) | A separate virtual network |
| Public subnet (`10.42.1.0/24`) | Network where the EC2 instance runs |
| Internet gateway and routing | Internet connectivity for the public subnet |
| Security group | Allows inbound **HTTP, TCP port 80** |
| EC2 (`t3.micro`) | Runs **Ubuntu 24.04 LTS** |
| EC2 User Data | Installs and starts **Nginx**, creates the web page |
| CloudFormation Outputs | Shows the website URL, instance ID, public IP and VPC ID |

**Security note:** There is no inbound SSH (port 22). This is a short HTTP-only learning exercise, not a production architecture.

## 3. Before you start

- [ ] Have an AWS account and permission to use CloudFormation, EC2, VPC, networking, and associated resources.
- [ ] Make sure your account can launch a `t3.micro` instance and that its billing/budget is understood.
- [ ] Keep both `README.md` and `cloudformation.yaml` on your computer **in the same folder**.
- [ ] Sign in to the [AWS Console](https://console.aws.amazon.com/).
- [ ] Pick **one AWS Region** for the entire lab, e.g. **US East (N. Virginia), `us-east-1`**. Use the same region for Dev and Test.
- [ ] Open [CloudFormation](https://console.aws.amazon.com/cloudformation/) and [EC2](https://console.aws.amazon.com/ec2/).

**Remember:** An AWS Region is selected near the upper-right corner of the AWS Console. If you cannot find a stack or EC2 instance later, first check the Region.

## 4. Part A - Create the Development environment

### Step A1 - Open the CloudFormation stack wizard

1. In the AWS Console, search for **CloudFormation** and open it.
2. In the left sidebar, select **Stacks** (if it is not already open).
3. Choose **Create stack** → **With new resources (standard)**.
4. On the template page, select **Choose an existing template**.
5. Under **Specify template**, select **Upload a template file**.
6. Choose **Choose file** and select **`cloudformation.yaml`** from the lab folder.
7. Select **Next**.

> You do **not** need to find a **Template** tab or a **Copy to clipboard** button. Always use the original YAML file provided with this lab.

### Step A2 - Set the Development values

On the **Specify stack details** page, enter:

| Field | Value |
|---|---|
| **Stack name** | `bongodev-ubuntu-nginx-dev` |
| **Environment** | `dev` |
| **LatestUbuntuAmiId** | Leave the provided default unchanged |

1. Select **Next**.
2. On **Configure stack options**, leave the defaults and select **Next**.
3. On **Review and create**, review the values, then select **Submit**.

The template does **not** create IAM roles or users, so an IAM-capabilities acknowledgement is not normally required for this lab.

### Step A3 - Watch CloudFormation create AWS resources

1. Open the newly created stack, `bongodev-ubuntu-nginx-dev`.
2. Select the **Events** tab; refresh as needed.
3. Notice resource states such as `CREATE_IN_PROGRESS` and `CREATE_COMPLETE`.
4. When the stack is `CREATE_COMPLETE`, open the **Resources** tab.
5. Locate resources such as `DemoVPC`, `PublicSubnet`, `WebSecurityGroup`, and `UbuntuNginxServer`.

**Checkpoint:** Can you name at least three AWS resources that CloudFormation created without you configuring them separately?

### Step A4 - Verify the EC2 instance

1. Open **EC2 → Instances** in the **same Region**.
2. Find the instance whose **Name** starts with `bongodev-ubuntu-nginx-dev` and ends with `-dev-nginx`.
3. Click its **Instance ID** to inspect the details.
4. Confirm:
   - Instance type: `t3.micro`
   - State: `Running` (once startup completes)
   - A **Public IPv4 address** is listed
   - The Security Group has inbound HTTP/TCP **port 80**
5. Optionally open **VPC → Your VPCs** to find the VPC with a name starting with the Dev stack name.

**Question:** Did you manually click **Launch instance** in EC2? **No.** CloudFormation created it from the YAML file.

### Step A5 - Open the Nginx web page

1. Return to **CloudFormation → Stacks → `bongodev-ubuntu-nginx-dev`**.
2. Open the **Outputs** tab.
3. Locate **`WebsiteURL`**. It should look similar to `http://3.123.45.67` (your IP will differ).
4. Open the URL in a new browser tab. Use **`http://`**, not `https://`.
5. Verify the page contains:

```text
Hello from bongoDev!
Infrastructure as Code is working!
Ubuntu 24.04 LTS + Nginx + AWS CloudFormation
Environment: dev
```

**Important:** A CloudFormation stack can reach `CREATE_COMPLETE` before Ubuntu finishes running the installation script (EC2 User Data). If the page is temporarily unavailable, investigate the instance status and the troubleshooting section below.

**Checkpoint:** Record your Dev instance ID, VPC ID and website URL in the table in Section 6.

## 5. Part B - Create a separate Testing environment from the SAME template

This is the main **IaC repeatability** exercise.

### Step B1 - Start a NEW stack (do not update Dev)

1. Keep the Development stack. **Do not select Update on the Dev stack.**
2. In **CloudFormation → Stacks**, select **Create stack → With new resources (standard)**.
3. Select **Choose an existing template** → **Upload a template file**.
4. Upload the **same `cloudformation.yaml` file** used in Part A. Do **not** modify or duplicate the YAML.
5. Select **Next**.

### Step B2 - Change only the stack name and environment

| Field | Development (already running) | Testing (new stack) |
|---|---|---|
| **Stack name** | `bongodev-ubuntu-nginx-dev` | `bongodev-ubuntu-nginx-test` |
| **Environment** | `dev` | `test` |
| **LatestUbuntuAmiId** | Default | Default |
| **YAML file** | `cloudformation.yaml` | **Same `cloudformation.yaml`** |

1. Enter `bongodev-ubuntu-nginx-test` as the stack name.
2. Set `Environment` to **`test`**.
3. Leave `LatestUbuntuAmiId` unchanged.
4. Select **Next → Next → Submit**.
5. When the stack reaches `CREATE_COMPLETE`, open **Outputs → WebsiteURL**.
6. Visit its URL and check that it says **`Environment: test`**.

### Step B3 - Compare Dev and Test

1. Go to **CloudFormation → Stacks**. You should see **two distinct stacks**.
2. Open the **Outputs** tab for each stack and compare instance IDs, VPC IDs, public IP addresses and website URLs.
3. Open **EC2 → Instances** to see the two Ubuntu machines.
4. Fill in the comparison table in Section 6.

**Why this matters:** Both environments have the same architecture defined in code, but independent infrastructure and different environment settings. Their VPC CIDR ranges happen to be the same (`10.42.0.0/16`), which is okay **because the VPCs are isolated**. Connected VPCs would need nonoverlapping ranges for many networking scenarios.

**A note about “identical”:** The template uses Canonical's **current** Ubuntu AMI and installs Nginx without pinning a package version. Deployments on different dates might get different AMIs or packages. Exact software-version matching requires pinning versions.

## 6. Lab evidence - complete this table

| Check | Development | Testing |
|---|---|---|
| Stack name | `bongodev-ubuntu-nginx-dev` | `bongodev-ubuntu-nginx-test` |
| Environment shown on website | `dev` | `test` |
| EC2 instance ID | __________________ | __________________ |
| VPC ID | __________________ | __________________ |
| Public IPv4 | __________________ | __________________ |
| Website URL | __________________ | __________________ |
| Stack status | __________________ | __________________ |

**Suggested screenshots (before cleanup):**

1. CloudFormation showing both stack names.
2. EC2 console showing both instances.
3. Dev webpage (`Environment: dev`) and Test webpage (`Environment: test`).

Avoid publishing unnecessary account identifiers or public IPs when sharing screenshots online.

## 7. Part C - CLEANUP: Delete BOTH environments

**Do not skip this step. AWS resources in this lab can incur charges.**

### Delete the Testing stack

1. Open **CloudFormation → Stacks**.
2. Select **`bongodev-ubuntu-nginx-test`**.
3. Choose **Delete**, then confirm.
4. Check **Events** and verify that deletion succeeds. Deleted stacks usually disappear from the active-stack list; they may remain visible in stack-history filters.

### Delete the Development stack

1. Return to the active **Stacks** list.
2. Select **`bongodev-ubuntu-nginx-dev`**.
3. Choose **Delete**, then confirm.
4. Verify deletion completes without `DELETE_FAILED`.

### Verify cleanup

- [ ] Neither Dev nor Test appears in the **active** CloudFormation stacks list.
- [ ] Both demo EC2 instances are terminated (they might remain visible as terminated temporarily).
- [ ] Both demo VPCs and security groups have been deleted.
- [ ] Neither website URL remains accessible once the instances are terminated.

If deletion fails, read the CloudFormation **Events** error message and resolve the blocker. Do not assume charges have stopped until billable resources are actually removed.

## 8. Troubleshooting

| Problem | Check / action |
|---|---|
| Can't find my stack or EC2 | Confirm you are in the **same AWS Region** used during creation. |
| Upload/validation error | Upload the **original `cloudformation.yaml`**, not this README; don't paste YAML into the wrong field. |
| `CREATE_FAILED` / rollback | Open the **Events** tab; read the first failure reason. Check AWS permissions, EC2 quota/capacity, and Ubuntu AMI parameter availability. |
| `WebsiteURL` shows a connection error | Ensure the EC2 instance is running, it has a public IPv4 address, and the URL begins with **`http://`**. Ubuntu may still be installing Nginx. |
| Timeout despite a running instance | Check the Security Group allows inbound TCP/80 and that the public subnet has a default route to the Internet Gateway. |
| Browser displays default Nginx page | User Data might not have finished or might have failed while creating `index.html`. Investigate bootstrap logs. |
| Can't SSH to the instance | Expected: this lab provides neither an SSH key pair nor inbound port 22. SSH is **not required** for the exercise. |
| Stack won't delete | Review the `DELETE_FAILED` resource in **Events** and follow the status reason. |

On an instance you are authorized to access through an approved administrative method, common Linux bootstrap logs are `/var/log/cloud-init-output.log` and `/var/log/bongodev-bootstrap.log`. EC2 **Actions → Monitor and troubleshoot → Get system log** may show boot information but not the complete application log.

## 9. Optional: Repeat using AWS CLI (advanced students)

**Skip this section if you're just learning the AWS Console.** These commands require a configured AWS CLI and access to `cloudformation.yaml` in the terminal's current working directory. For AWS CloudShell, upload the YAML file there first.

```bash
# Check which AWS account you are using
aws sts get-caller-identity

# Create or update the Dev stack
aws cloudformation deploy \
  --template-file cloudformation.yaml \
  --stack-name bongodev-ubuntu-nginx-dev \
  --parameter-overrides Environment=dev \
  --region us-east-1

# Create or update the Test stack from the SAME file
aws cloudformation deploy \
  --template-file cloudformation.yaml \
  --stack-name bongodev-ubuntu-nginx-test \
  --parameter-overrides Environment=test \
  --region us-east-1

# Show Test stack outputs
aws cloudformation describe-stacks \
  --stack-name bongodev-ubuntu-nginx-test \
  --query 'Stacks[0].Outputs' \
  --region us-east-1

# Delete both stacks
aws cloudformation delete-stack \
  --stack-name bongodev-ubuntu-nginx-test \
  --region us-east-1

aws cloudformation delete-stack \
  --stack-name bongodev-ubuntu-nginx-dev \
  --region us-east-1

# Confirm each stack has finished deleting
aws cloudformation wait stack-delete-complete \
  --stack-name bongodev-ubuntu-nginx-test \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name bongodev-ubuntu-nginx-dev \
  --region us-east-1
```

**Tip:** Using `deploy` with an existing stack name **updates** that stack. Using a **new stack name** creates a new stack. This is why the Dev and Test stack names must be different.

## 10. Knowledge check

1. What problem does IaC solve compared with manual infrastructure setup?
2. What are the `Parameters`, `Resources`, and `Outputs` sections of CloudFormation?
3. What is an AWS CloudFormation **stack**?
4. Why are there two EC2 instance IDs even though we uploaded the same template twice?
5. Why does Ubuntu's Nginx installation use `apt-get`, not `dnf`?
6. What is **Canonical**, and why is its AWS Systems Manager public AMI parameter used?
7. Does `CREATE_COMPLETE` guarantee Nginx is ready? Why or why not?
8. What must you delete after this lab, and why?

## 11. Resources and important notes

- [AWS: Create a stack in the CloudFormation console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html)
- [Canonical: Create CloudFormation templates using Ubuntu AMIs](https://ubuntu.com/aws/docs/aws-how-to/instances/build-cloudformation-templates/)
- [AWS: CloudFormation walkthrough](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/gettingstarted.walkthrough.html)

**Costs:** An EC2 instance, EBS volume, public IPv4 address, and network traffic may be billable depending on your account, free-tier eligibility, and region. Two stacks mean two sets of resources. You are responsible for checking and deleting yours.

**Production disclaimer:** This lab serves a public HTTP page for training. Real workloads normally need HTTPS, appropriate access restrictions, monitoring, patching, and other safeguards.

---

**bongoDev - Learn. Practice. Prove you're ready.**
