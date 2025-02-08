# Ansible + GitHub Actions CI/CD Lab :rocket:

> A simple and straightforward guide on how to set up an **Ansible + GitHub Actions** pipeline to install NGINX on AWS EC2 instances and deploy a basic website. 

<br/>

## :page_facing_up: Table of Contents
1. [Introduction](#introduction)
2. [Repository Structure](#repository-structure)
3. [Prerequisites](#prerequisites)
4. [Setup Instructions](#setup-instructions)
5. [File Explanations](#file-explanations)
   - [GitHub Actions Workflow (`.github/workflows/cicd.yml`)](#1-github-actions-workflow)
   - [Ansible Dynamic Inventory (`aws_ec2.yml`)](#2-ansible-dynamic-inventory)
   - [Playbook to Install NGINX (`nginx.yml`)](#3-playbook-to-install-nginx)
   - [Simple HTML Page (`index.html`)](#4-simple-html-page)
6. [Usage](#usage)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)
9. [Contributing](#contributing)
10. [License](#license)

<br/>

## Introduction
This lab demonstrates how to use GitHub Actions to automatically deploy a simple NGINX server and a static HTML page onto AWS EC2 instances. The core concepts include:

- **Infrastructure as Code (IaC):** Using Ansible to define and manage the state of your servers.
- **Continuous Integration/Continuous Deployment (CI/CD):** Using GitHub Actions to automate the testing and deployment process.
- **Dynamic Inventory:** Automatically discovering AWS EC2 instances using Ansible’s `aws_ec2` plugin.

<br/>

## Repository Structure
```
.
├── .github
│   └── workflows
│       └── cicd.yml       # GitHub Actions pipeline configuration
├── aws_ec2.yml            # Ansible dynamic inventory configuration
├── index.html             # Basic HTML page served by NGINX
├── nginx.yml              # Ansible playbook to install NGINX and deploy index.html
└── README.md              # The file you're reading now
```

> **Note:** The `.git` folder and other internal files are excluded from the repository for security and clarity. Never commit private/sensitive data to your public repository.

<br/>

## Prerequisites

1. **GitHub Account**: With the ability to create repositories.
2. **AWS Account**: IAM user or role with EC2 permissions (create, list, etc.).
3. **AWS Credentials**: 
   - AWS Access Key ID
   - AWS Secret Access Key
4. **SSH Key Pair**: A key pair that your EC2 instances trust. You will store the **private key** as a secret in GitHub.
5. **EC2 Instances**: Ensure they have a matching **tag** (or region) to be discovered via the dynamic inventory in `aws_ec2.yml`.
6. **Basic Knowledge**: Familiarity with Git, GitHub, and AWS.

<br/>

## Setup Instructions

1. **Create a New Repository**: 
   - Create a repository in your GitHub account (or fork/clone an existing one).
   - Clone it to your local machine.

2. **Configure GitHub Secrets** (in your repository’s **Settings > Secrets and variables > Actions**):
   - **AWS_ACCESS_KEY_ID**: Your AWS Access Key ID.
   - **AWS_SECRET_ACCESS_KEY**: Your AWS Secret Access Key.
   - **PRIVATE_KEY**: The **private** portion of your SSH key. (Ensure the corresponding **public** key is on your EC2 instances.)

3. **Add/Copy the Necessary Files**:
   - `.github/workflows/cicd.yml`
   - `aws_ec2.yml`
   - `nginx.yml`
   - `index.html`
   - `README.md` (this file)

4. **Commit and Push** to your main (or default) branch:
   ```bash
   git add .
   git commit -m "Add CI/CD pipeline with Ansible"
   git push origin main
   ```

5. **Check** your GitHub Actions tab to see the pipeline run.

<br/>

## File Explanations

### 1. GitHub Actions Workflow (`.github/workflows/cicd.yml`)
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Check out the repository code
      - name: Check out code
        uses: actions/checkout@v2

      # Step 2: Set up Python environment
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'

      # Step 3: Update apt repositories
      - name: Update apt repositories
        run: sudo apt update

      # Step 4: Install Boto3 (AWS SDK for Python)
      - name: Install Boto3
        run: pip install boto3

      # Step 5: Install Ansible
      - name: Install Ansible
        run: pip install ansible

      # Step 6: Configure AWS credentials
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2  # Change to your AWS region

      # Step 7: List EC2 servers via dynamic inventory (for debugging)
      - name: List EC2 servers
        run: ansible-inventory --graph -i aws_ec2.yml

      # Step 8: Print SSH key (optional debug step - remove or comment out if not needed)
      - name: Print SSH key
        run: echo "$PRIVATE_KEY"
        env:
          PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}

      # Step 9: Set up SSH directory and write private key
      - name: Set up SSH
        run: |
          mkdir -p ~/.ssh
          echo "$PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
        env:
          PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}

      # Step 10: Run Ansible playbook
      - name: Run Ansible Playbook
        run: ansible-playbook -i aws_ec2.yml nginx.yml
        env:
          PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
          ANSIBLE_HOST_KEY_CHECKING: False
```
**Key Points**:
- Uses `aws-actions/configure-aws-credentials@v4` to authenticate with AWS.
- Installs `boto3` and `ansible` to manage AWS resources and run Ansible playbooks.
- Leverages `ansible-inventory --graph` to dynamically list EC2 hosts before running `ansible-playbook`.

<br/>

### 2. Ansible Dynamic Inventory (`aws_ec2.yml`)
```yaml
plugin: aws_ec2
regions:
  - us-east-2
filters:
  tag:team:
    - julio
```
**Key Points**:
- **plugin:** Specifies we are using the `aws_ec2` plugin for dynamic inventory.
- **regions:** Lists the AWS regions you want to scan.
- **filters:** Only gather hosts that match specified tags (e.g., `team=julio`). Adjust to match your setup.

<br/>

### 3. Playbook to Install NGINX (`nginx.yml`)
```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Copy Website Files
      copy:
        src: index.html
        dest: /var/www/html/
        mode: '0644'
```
**Key Points**:
- Installs **NGINX** on all hosts in the dynamic inventory (`hosts: all`).
- Copies the `index.html` from your local repo to `/var/www/html` on the EC2 instance.

<br/>

### 4. Simple HTML Page (`index.html`)
```html
<h1>Hello Ansible CICD using GitHub Actions</h1>
```
**Key Points**:
- This is a placeholder file that demonstrates a basic webpage served by NGINX.

<br/>

## Usage
1. **Update** the region and filter tags in `aws_ec2.yml` to match your AWS environment.
2. **Push** your changes to the `main` branch.
3. **Monitor** the pipeline in the GitHub Actions tab.

<br/>

## Verification
1. Once the pipeline completes successfully, **SSH** into your EC2 instance and run:
   ```bash
   nginx -v
   ls /var/www/html/index.html
   ```
   - You should see the installed NGINX version and the `index.html` file.
2. **Web Browser**: Access the public IP (or DNS) of the EC2 instance in a browser:
   ```
   http://<your-ec2-public-ip-or-dns>/
   ```
   You should see the message: **“Hello Ansible CICD using GitHub Actions”**.

<br/>

## Troubleshooting
- **EC2 Instances Not Found**: Check the region and tag filters in `aws_ec2.yml`.
- **Permission Denied**: Ensure your SSH key matches the key on the EC2 instance.
- **Ansible Host Key Checking**: If you see host key mismatch errors, confirm `ANSIBLE_HOST_KEY_CHECKING=False` is set (as shown in the workflow).
- **Boto3/Ansible Versions**: If you encounter issues, try updating them in the pipeline steps.

<br/>

## Contributing
Contributions are welcome! Feel free to open **Issues** or submit **Pull Requests** to improve or extend the functionality.

<br/>

## License
This project is provided under the **MIT License**. For more details, see [LICENSE](LICENSE) (if added) or choose a suitable open-source license for your project.

---

**Happy Deploying!** :tada: 
If you have any suggestions or face any issues, please open an issue in this repository.