# eShopOnWeb_setupp

---

````md
# Jenkins CI/CD Pipeline — .NET Application Deployment on IIS (Windows)

This project is a Jenkins CI/CD pipeline for a **.NET (ASP.NET Core 8)** application.

- Build/Test happens on **Linux Jenkins**
- Deployment happens on **Windows IIS**
- A separate **Windows Jenkins Agent** is used for IIS deployment

Repo:
https://github.com/thegagankapoor/eShopOnWeb

---

## 1️⃣ Install Jenkins on Linux (Ubuntu)

### 1.1 Install Java 21
```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```


Verify:
```bash
java -version
````

# ### 1.2 Install Zip

# ```bash
# sudo apt install zip unzip -y
# ```

# Verify:

# ```bash
# zip --version
# ```

### 1.3 Install Jenkins

Install required packages first:
```bash
wget -O jenkins.war https://get.jenkins.io/war-stable/latest/jenkins.war
java -jar jenkins.war --httpPort=8080
```

Go into Security Group of the EC2 Instance:
```bash
Add Inbound Rule:
Port:8080
Source: MyIP
```
```bash
Add Inbound Rule:
Port:9000
Source: MyIP
```

### 1.4 Open Jenkins

```txt
http://EC2InstancePublicIP:8080
```




---

## 2️⃣ Unlock Jenkins (First Time Setup)

### 2.1 Get initial admin password

Run:

```bash
cat /home/ubuntu/.jenkins/secrets/initialAdminPassword
```

Copy the password and paste it in Jenkins UI.

---

## 3️⃣ Install Required Jenkins Plugins

Go to:

**Manage Jenkins → Plugins → Available Plugins**

Install:

* .NET SDK Support
* OWASP Dependency-Check
* SonarQube Scanner
* Pipeline Stage View

Restart Jenkins.

---

## 4️⃣ Configure .NET SDK in Jenkins (Linux Build) & OWASP Dependency-Check & SonarQube

Go to:

**Manage Jenkins → Tools → .NET SDK installations**

Add:

* Name: `dotnet-8`
* Install automatically: ✅
* .NET Version: `.NET 8`
* Release: 8.0.24
* SDK: 8.0.418
* Platform: `Linux x64`

Add: **Dependency-Check installations**

* Name: `DP-Check`
* Install automatically: ✅
* Add Installer: Install from github.com

Add: **SonarQube Scanner installations**

* Name: `SonarQube`
* Install automatically: ✅

---

## Installation of SonarQube

```bash
sudo adduser sonarqube

sudo usermod -aG sudo sonarqube

sudo su - sonarqube

sudo apt install zip unzip -y

wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.2.0.119303.zip

unzip sonarqube-26.2.0.119303.zip

 Set permissions

chmod -R 755 /home/sonarqube/sonarqube-26.2.0.119303
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-26.2.0.119303

Start
cd /home/sonarqube/sonarqube-26.2.0.119303/bin/linux-x86-64/
./sonar.sh start
```
---

## 5️⃣ Add GitHub & SonarQube Credentials in Jenkins

Go to:

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

Add:

* Kind: Secret text
* Secret: GitHub Personal Access Token (PAT)
* ID: `github-credentials`

Add:

* Kind: Secret text
* Secret: Your Sonar Token
* ID: `sonarqube-token`
---

## 5️⃣ How to Generate SonarQube Token

**My Account → Security**

* Token Name: jenkins-token
* Token Type: Global Analysis Token

---

**Manage Jenkins → System**
* Click on SonarQube Installations

* Name: `SonarQube`
* ServerURL: `http://localhost:9000`
* Server Authentication Token: `sonarqube-token`



## 6️⃣ Setup Windows Machine (For IIS Deployment)

This Windows machine will work as:

* Jenkins Agent
* IIS Server

---

## 7️⃣ Install IIS on Windows (GUI Method)

### 7.1 Open Windows Features

1. Press `Windows + R`
2. Type:

   ```txt
   optionalfeatures
   ```
3. Press Enter

### 7.2 Enable IIS

1. Scroll down and enable:
   ✅ Internet Information Services

### 7.3 Enable IIS Components

Expand **Internet Information Services** and enable:

✅ Web Management Tools

* IIS Management Console

✅ World Wide Web Services → Application Development Features

* ISAPI Extensions
* ISAPI Filters

✅ World Wide Web Services → Common HTTP Features

* Default Document
* Directory Browsing
* HTTP Errors
* Static Content

### 7.4 Install IIS

1. Click OK
2. Windows will install IIS
3. Click Close

### 7.5 Verify IIS

Open browser and go to:

```txt
http://localhost
```

You should see the IIS welcome page.

### 7.6 Open IIS Manager

1. Press `Windows + R`
2. Type:

   ```txt
   inetmgr
   ```
3. Press Enter

---

## 8️⃣ Install Java on Windows (Required for Jenkins Agent)

Install Java 17 on Windows.

Verify:

```powershell
java -version
```

---

## 9️⃣ Install ASP.NET Core Hosting Bundle on Windows

To run ASP.NET Core apps on IIS, you must install the Hosting Bundle.

Download and install:

* ASP.NET Core Hosting Bundle (.NET 8) (Windows x64)

After installation, restart IIS:

```powershell
iisreset
```

Verify runtime:

```powershell
dotnet --list-runtimes
```

---

## 🔟 Create Windows Jenkins Agent Node

Go to:

**Manage Jenkins → Nodes → New Node**

Create:

* Name: `windows-iis-agent`
* Type: Permanent Agent
* Number of executors: `1`
* Remote root directory:

  ```txt
  C:\Jenkins
  ```
* Labels:

  ```txt
  windows-iis-agent
  ```
* Usage:

  ```txt
  Only build jobs with label expressions matching this node
  ```
* Launch method:

  ```txt
  Launch agent by connecting it to the controller
  ```

Save.

---

## 1️⃣1️⃣ Connect Windows Agent to Jenkins

### 11.1 Create Jenkins folder

```powershell
mkdir C:\Jenkins
cd C:\Jenkins
```

### 11.2 Download agent.jar

Replace `<JENKINS-IP>` with your Linux Jenkins IP:

```powershell
curl.exe -sO http://<JENKINS-IP>:8080/jnlpJars/agent.jar
```

### 11.3 Run the agent

Copy the command from Jenkins Node page.

Example:

```powershell
java -jar agent.jar -url http://<JENKINS-IP>:8080/ -secret <SECRET> -name "windows-iis-agent" -webSocket -workDir "C:\Jenkins"
```

---

## 1️⃣2️⃣ Create Jenkins Pipeline Job

Go to Jenkins:

**New Item → Pipeline**

Name:

```txt
eShopOnWeb-Pipeline
```

Select:

Pipeline script from SCM → Git

Repo URL:

```txt
https://github.com/thegagankapoor/eShopOnWeb
```

Branch:

```txt
main
```

Script path:

```txt
Jenkinsfile
```

Save.

---

## 1️⃣3️⃣ Add Jenkinsfile to Repo

Create a file named:

```txt
Jenkinsfile
```

Paste your final working Jenkins pipeline code.

---

## 1️⃣4️⃣ Run the Pipeline

Go to Jenkins job and click:

✅ Build Now

---

## 1️⃣5️⃣ Open Application in Browser

Open on Windows machine:

```txt
http://localhost:8081
```

---

## 1️⃣6️⃣ Common Errors

### dotnet --version fails (Windows)

Reason:

* Windows has Runtime installed, not SDK

Fix:
Use:

```powershell
dotnet --list-runtimes
```

---

### SQL Server connection error after deployment

Reason:

* Application requires SQL Server configuration

Example error:

```txt
SqlException: error 26 - Error Locating Server/Instance Specified
```

This is expected if SQL Server is not configured.

---

```
```
