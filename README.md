# Self-Hosted Jenkins CI/CD Pipeline on AWS EC2

This project demonstrates how to build a self-hosted Continuous Integration and Continuous Delivery (CI/CD) pipeline using Jenkins on an AWS EC2 instance. The pipeline integrates with GitHub, installs dependencies, runs tests, builds a Node.js application, and deploys the generated static files to Nginx.

## Project Architecture

```text
Developer
   |
   | Git Push
   v
GitHub Repository
   |
   | Source Code Checkout
   v
Jenkins on AWS EC2 (Ubuntu 24.04)
   |
   |-- Checkout
   |-- Install Dependencies
   |-- Run Tests
   |-- Build Application
   |-- Deploy Application
   |
   v
Nginx Web Server
   |
   v
Application Available in Browser
```

## 1. Prerequisites and Infrastructure Setup

### Required Infrastructure

- AWS account
- Ubuntu 24.04 LTS EC2 instance
- GitHub repository containing a Node.js application
- Jenkins installed on EC2
- Java 21 or later, supported by the installed Jenkins release
- Git
- Node.js and npm
- Nginx web server

### Recommended EC2 Configuration

- Instance type: `t3.medium` for a learning environment
- Storage: 20 GB or more, depending on build requirements
- Security group:
  - SSH (22): allow only your trusted IP address.
  - HTTP (80): allow access if the deployed website should be public.
  - Jenkins (8080): initially allow only your trusted IP address.

**Security note:** Do not expose the Jenkins administration interface publicly to everyone. For production, prefer private access through a VPN, a bastion host, or an appropriately configured HTTPS reverse proxy.

Connect to the EC2 instance using SSH:

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace `your-key.pem` and `YOUR_EC2_PUBLIC_IP` with your actual values.

## 2. Update Ubuntu Packages

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic utilities:

```bash
sudo apt install -y curl wget gnupg ca-certificates fontconfig
```

## 3. Install Java 21

Jenkins requires a supported Java runtime. For the Ubuntu 24.04 environment, install OpenJDK 21:

```bash
sudo apt install -y openjdk-21-jre
```

Verify the installation:

```bash
java -version
```

Check the Java executable:

```bash
which java
```

## 4. Install Git

Install Git for checking out the application source code from GitHub.

```bash
sudo apt install -y git
```

Verify:

```bash
git --version
```

## 5. Install Jenkins on Ubuntu 24.04

### Add the official Jenkins signing key

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

### Add the Jenkins LTS repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### Install Jenkins

```bash
sudo apt update
sudo apt install -y jenkins
```

These commands use the official Jenkins Debian/Ubuntu LTS package repository.

## 6. Start and Enable Jenkins

Enable Jenkins to start automatically after a reboot:

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Check the service status:

```bash
sudo systemctl status jenkins
```

Check whether Jenkins is listening on port 8080:

```bash
sudo ss -lntp | grep 8080
```

If the service fails to start, inspect its logs:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

## 7. Unlock Jenkins

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password.

Open the following address in your browser:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Enter the initial administrator password.

Complete the initial configuration:

1. Select **Install suggested plugins**.
2. Create the first administrator account.
3. Configure the Jenkins URL.
4. Save and finish the setup.

Keep Jenkins updated and install only plugins required for your pipeline.

## 8. Install Node.js and npm

The example application uses Node.js. Install a supported Node.js LTS release using an appropriate trusted package source.

For example, to use the NodeSource 22.x repository:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify the versions:

```bash
node --version
npm --version
```

Node.js 22 is an example runtime choice. Select a supported version compatible with your application's `package.json` and dependencies.

## 9. Install and Configure Nginx

Install Nginx as the web server for the deployed static application:

```bash
sudo apt install -y nginx
```

Enable and start the service:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Verify its status:

```bash
sudo systemctl status nginx
```

Test the default page by opening:

```text
http://YOUR_EC2_PUBLIC_IP
```

Ensure that inbound TCP port 80 is allowed in the EC2 security group if you want the website to be accessible over the internet.

## 10. Prepare the GitHub Repository

Create or use a GitHub repository containing your Node.js application.

A typical repository structure is:

```text
my-node-app/
├── package.json
├── package-lock.json
├── src/
├── public/
├── tests/
├── dist/
└── Jenkinsfile
```

The exact structure depends on your framework. For this example, the application must provide:

- `package.json` with an `npm ci`-compatible lockfile.
- An `npm test` script.
- An `npm run build` script that generates a `dist/` directory.
- Static files inside `dist/` that Nginx can serve.

If your project uses another build output directory, update the deployment stage accordingly.

### Example package.json scripts

Your project should define the relevant scripts. For example:

```json
{
  "scripts": {
    "test": "node --test",
    "build": "mkdir -p dist && cp -r public/. dist/"
  }
}
```

This is a minimal example for a project whose static website files are stored in `public/`. For React, Vue, Angular, or another framework, use the build command provided by that framework.

Commit and push the application to GitHub.

## 11. Configure GitHub Access

### Public repository

For a public repository, Jenkins can check out source code using the HTTPS repository URL without credentials.

Example:

```text
https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
```

### Private repository

For a private repository, configure appropriate GitHub credentials in Jenkins.

Recommended approach:

1. Create a GitHub fine-grained personal access token with only the required repository permissions, or configure a suitable deploy key.
2. In Jenkins, open **Manage Jenkins → Credentials**.
3. Add the credential to the appropriate store.
4. Use a suitable credential type for the selected Git checkout method.
5. Reference the credential ID in the pipeline.

Never store access tokens or passwords directly in the `Jenkinsfile` or commit them to GitHub.

## 12. Configure Jenkins Plugins

From the Jenkins dashboard, navigate to:

**Manage Jenkins → Plugins**

Ensure that the required plugins are installed:

- Pipeline
- Git
- Credentials Binding, if the pipeline uses bound credentials
- Pipeline Stage View, if you want the stage visualization

Some of these plugins may already be installed through the suggested plugins setup.

Restart Jenkins if the plugin installation process requests it.

## 13. Configure Deployment Directory Permissions

For this learning example, Jenkins will publish the generated static files to Nginx's web root.

Create the directory and assign ownership to the Jenkins service account:

```bash
sudo mkdir -p /var/www/html
sudo chown -R jenkins:jenkins /var/www/html
sudo chmod 755 /var/www/html
```

Nginx can read files that Jenkins publishes into this directory, provided the files have appropriate read permissions.

This setup is intended for a single-instance learning project. For production, separate the Jenkins controller from the deployment target and use a restricted deployment identity.

## 14. Create the Jenkinsfile

Create a file named `Jenkinsfile` in the root of your GitHub repository.

The following declarative pipeline checks out the source code, installs dependencies, runs tests, builds the application, and publishes static files to Nginx.

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    set -eu
                    git --version
                    node --version
                    npm --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -eu
                    npm ci
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -eu
                    npm test
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                    set -eu
                    npm run build
                    test -d dist
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    test -n "$(find dist -type f -print -quit)"

                    find dist -type f -exec chmod 644 {} \\;
                    find dist -type d -exec chmod 755 {} \\;

                    cp -r dist/. /var/www/html/

                    echo "Deployment completed successfully."
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
```

### Important notes

- `checkout scm` checks out the revision associated with the configured Pipeline job.
- `npm ci` requires a compatible `package-lock.json` or other supported npm lockfile.
- `npm test` must exist in `package.json`.
- `npm run build` must generate `dist/`.
- The deployment stage copies files into `/var/www/html/`.
- The pipeline does not delete old destination files. If you need a clean deployment, use a carefully configured synchronization command after validating the source and destination paths.

If your project uses a different build directory, change `dist` to the actual directory.

## 15. Create a Pipeline Job in Jenkins

Open the Jenkins dashboard.

1. Click **New Item**.
2. Enter a job name, such as `nodejs-cicd-pipeline`.
3. Select **Pipeline**.
4. Click **OK**.
5. Scroll to the Pipeline section.
6. Set **Definition** to **Pipeline script from SCM**.
7. Select **Git** as the SCM.
8. Enter your GitHub repository URL.
9. Add credentials if the repository is private.
10. Set the branch to the branch containing your `Jenkinsfile`, such as `*/main`.
11. Set the Script Path to `Jenkinsfile`.
12. Click **Save**.

## 16. Run the CI/CD Pipeline

Open your newly created Jenkins job and click **Build Now**.

Select the build number, then open **Console Output**.

The pipeline should execute these stages in sequence:

1. Checkout
2. Verify Tools
3. Install Dependencies
4. Test
5. Build
6. Deploy

A successful run should show the message:

```text
CI/CD pipeline completed successfully.
```

The application files should be present under:

```text
/var/www/html/
```

Check the files on EC2:

```bash
ls -la /var/www/html/
```

Test the website:

```text
http://YOUR_EC2_PUBLIC_IP
```

## 17. Test Automatic CI/CD Updates

After verifying the initial build:

1. Modify a file in your GitHub repository.
2. Commit and push the changes.
3. Trigger a new Jenkins build.
4. Check the build logs.
5. Open the website and verify the updated content.

To trigger builds automatically after a GitHub push, configure a GitHub webhook.

### Configure a GitHub webhook

In your GitHub repository:

1. Open **Settings → Webhooks**.
2. Click **Add webhook**.
3. Enter the webhook URL:

```text
http://YOUR_JENKINS_PUBLIC_IP:8080/github-webhook/
```

4. Choose `application/json` as the content type.
5. Select the push event.
6. Save the webhook.

In Jenkins, enable the appropriate GitHub webhook trigger for the job if required by your plugin and job configuration.

**Security:** Do not expose an unsecured Jenkins instance just to receive webhooks. Use HTTPS, appropriate access controls, and a restricted network design. A private Jenkins controller may require a secure webhook relay or another supported integration method.

If you do not configure a webhook, you
