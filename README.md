
## Step-By-Step Guide

# Overview 
This project automates the deployment of a WordPress application using Dokku (a self-hosted PaaS) on a local WSL/Ubuntu environment, with CI/CD pipelines, monitoring via Prometheus/Grafana, and security hardening.

## Step 1: Initial Setup

1. **Update WSL**  
   Run the following commands to update your WSL and install necessary dependencies:  
   ```bash
   sudo apt update && sudo apt upgrade -y  
   sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
   ```

2. **Set Hostname in WSL**  
   Open and edit the `/etc/hosts` file in WSL:  
   ```bash
   sudo nano /etc/hosts
   ```  
   Add the following entry:  
   ```
   127.0.0.1 devops.s14.local
   ```  
   > **Note**: This is a secondary setup. The actual setup happens in the Windows `hosts` file at `C:\Windows\System32\drivers\etc\hosts` for accessing `devops.s14.local` from your browser.

### ✅ Step-by-Step: Editing the Windows Hosts File

1. **Open Notepad as Administrator**  
   - Press `Win + S` and search for **Notepad**.  
   - Right-click on Notepad and select **Run as administrator**.

2. **Open the Hosts File**  
   - In Notepad, go to `File > Open`.  
   - Navigate to:  
     ```
     C:\Windows\System32\drivers\etc\
     ```  
   - In the file picker, change the file type from **Text Documents** to **All Files (*.*)** to see the `hosts` file.  
   - Select `hosts` and click **Open**.

3. **Add Custom Domain Mapping**  
   At the bottom of the file, add the following line:  
   ```
   127.0.0.1 devops.s14.local
   ```  

4. **Save the File**  
   Press `Ctrl + S` to save. Admin rights are required to save changes to this file.

### ✅ Test it:  
Open your browser and visit `http://devops.s14.local`. It should now work!  
After saving and closing the file, ping the address from the command prompt to confirm it is functioning:  
```bash
ping devops.s14.local
```

## Step 2: Dokku Installation and Configuration

1. **Install Dokku**  
   Download and install Dokku:  
   ```bash
   wget -NP . https://dokku.com/install/v0.35.9/bootstrap.sh  
   sudo DOKKU_TAG=v0.35.9 bash bootstrap.sh
   ```

2. **Install LetsEncrypt Plugin**  
   ```bash
   sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
   ```

3. **Check Versions**  
   Verify the installation by checking versions:  
   ```bash
   docker --version  
   dokku --version
   ```

4. **Download WordPress**  
   ```bash
   wget https://wordpress.org/latest.tar.gz  
   tar -xvzf latest.tar.gz  
   mv wordpress wordpress-app  
   cd wordpress-app
   ```

5. **Create Dokku App**  
   ```bash
   dokku apps:create wordpress-app
   ```

6. **Setup Nginx Domain**  
   ```bash
   dokku domains:add wordpress-app devops.s14.local
   ```

7. **Install MySQL**  
   WordPress requires a database. Install MySQL:  
   ```bash
   sudo dokku plugin:install https://github.com/dokku/dokku-mysql.git mysql  
   dokku mysql:create wordpress-db  
   dokku mysql:link wordpress-db wordpress-app
   ```

8. **Deploy WordPress via Dockerfile**  
   Add WordPress app to the Dockerfile to enable Dokku access.

# Errors I encountered in  Step 2:
#Dokku Installation and Configuration

1. **Git Error: Initializing Git Repository**  
   Error:  
   ```
   error: src refspec main does not match any
   error: failed to push some refs to 'devops.s14.local:wordpress-app'
   ```  
   **Solution**:  
   Change local branch to `main`:  
   ```bash
   git branch -m master main
   ```

2. **SSH Connection Refused Error**  
   Error:  
   ```
   ssh: connect to host devops.s14.local port 22: Connection refused
   fatal: Could not read from remote repository.
   ```  
   **Solution**:  
   Upgrade Dokku to version 0.35.18, install `openssh-server`, and enable SSH:  
   ```bash
   sudo apt install openssh-server  
   sudo systemctl enable ssh  
   sudo systemctl start ssh
   ```

3. **Git Push Works**  
   After resolving the SSH connection issue, run:  
   ```bash
   git push dokku main:main
   ```

4. **SSL Issue: Cannot Use Let's Encrypt for Local Domains**  
   **Solution**:  
- Use a CA certificate for local domains:  
- Create a local Certificate Authority (CA)
- Generate a cert for devops.s14.local and sign it with your custom CA
- Apply the certificate to your Dokku app - $ dokku certs:add wordpress-app /tmp/devops.s14.local.crt /tmp/devops.s14.local.key

**Error**-  !     CRT file specified not found, please check file paths
**Solution** - moved crt and .key files to a tmp folder 
changed permissions(chmod 600), and restarted, before i could add
- Copy the certificate to the system’s trusted certificate director
sudo mv /tmp/devops.s14.local.crt /usr/local/share/ca-certificates/
- Update the CA certificates - sudo update-ca-certificates
- Verify certificate addition - ls /etc/ssl/certs/

App is still accessible at -   https://devops.s14.local
     ```

## Step 3: CI/CD Setup

1. **Create CI/CD Workflow**  
   Create a `.github/workflows/deploy.yml` file in your GitHub repository.

2. **Add SSH Key to GitHub**  
   - Generate `id_rsa` and check it with `cat ~/.ssh/id_rsa.pub`.  
   - Add this key to GitHub repo secrets under **Settings > Secrets and variables > Actions**.

3. **Roll Back Deployment on Failure**  
   Add rollback logic to the CI/CD YAML file:  
   ```yaml
   - name: Push to Dokku
     id: dokku-deploy
     run: |
       set -e
       if ! git push dokku main; then
         echo "Deployment failed! Rolling back to previous release..."
         ssh -o StrictHostKeyChecking=no dokku@devops.s14.local "dokku releases:revert wordpress-app \$(dokku releases:list wordpress-app --no-truncate | sed -n '3p' | awk '{print \$1}')"
         exit 1
       fi
   ```
4. **Possible errors in Step 3: CI/CD Setup**-
   - make sure token has workflow permissions, had to create a new token and update credentials

  **Self-Hosted Runner**  
   If GitHub Actions cannot resolve `devops.s14.local`because of local machine choice , use a self-hosted runner:  
   - Go to **Settings > Actions > Runners**, click "New self-hosted runner", and follow the prompts.  
   - Start the runner with:  
     ```bash
     ./run.sh
     ```

   Update your workflow to use the self-hosted runner:  
   ```yaml
   runs-on: self-hosted
   ```
   - documentation - https://docs.github.com/en/actions/hosting-your-own-runners

  **Remote Dokku Already Exists**
   - Run git remote add dokku dokku@devops.s14.local:wordpress-app
  git remote add dokku dokku@devops.s14.local:wordpress-app
  shell: /usr/bin/bash -e {0}
  env:
    SSH_AUTH_SOCK: /tmp/ssh-XXXXXXEEXZev/agent.401159
    SSH_AGENT_PID: 401160
error: remote dokku already exists.
Error: Process completed with exit code 3.


**Solution** - I added this to the yaml file
- name: Check if dokku remote exists
        run: |
          if ! git remote get-url dokku; then
            git remote add dokku dokku@devops.s14.local:wordpress-app
          fi



## Step 4: Monitoring and Alerts

1. **Install Prometheus**  
   - Install Prometheus and configure it to scrape Node Exporter metrics.
```cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.51.1/prometheus-2.51.1.linux-amd64.tar.gz
sudo tar xvf prometheus-2.51.1.linux-amd64.tar.gz
sudo mv prometheus-2.51.1.linux-amd64 prometheus
```

```sudo nano /etc/systemd/system/prometheus.service
```

```[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=root
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data

[Install]
WantedBy=default.target

```
2.  **NodeExporter**
 ```
 cd /opt
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
sudo tar xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64 node_exporter
 ```

 ```sudo nano /etc/systemd/system/node_exporter.service
 ```

 ```[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
ExecStart=/opt/node_exporter/node_exporter

[Install]
WantedBy=default.target
```
3. **Configure Prometheus to scrape NodeExporter** 
  ```sudo nano /opt/prometheus/prometheus.yml ```
add this to scrap_config in /opt/prometheus/prometheus.yml
  ```  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
   ```

   check if prometheus is scrapping at - http://localhost:9090/targets


4. **Install Grafana**  
   Install Grafana for visualization of metrics
   ```sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install grafana
``` 
   
   ➡️ http://localhost:3000 (Default credentials: admin / admin). It will require you to create a new password immediately


5. **Configure Prometheusand Grafana to montior
CPU usage
Memory usage
Disk Usage
Application uptime and
Create Alert rules 

1. **Add Prometheus as a Data Source**

- From the left sidebar, click on **⚙️ (Gear icon) > Data Sources**.
- Click **“Add data source”**.
- Select **Prometheus**.
- In the **URL** field, enter:
   ```
   http://localhost:9090
   ```
   
---

2. **Import a Node Exporter Dashboard**

- From the sidebar, click the **+ (plus icon) > Import**.
- In the **Import via grafana.com** field, enter:
   ```
   1860
   ```
   and click **Load**.
- Choose Prometheus as the data source.
- Click **Import**.

You’ll now see a beautiful dashboard showing:
- CPU
- Memory
- Disk
- Network
- Uptime

---

3. **Create a CPU Alert Rule**

Create a simple CPU usage alert:
- Go to **Alerting > Alert Rules > + New alert rule**.
- Set name: `High CPU Usage Alert`.

3. Under **Query A**:
   ```promql
   avg(rate(node_cpu_seconds_total{mode="user"}[1m])) * 100
   ```

4. Set condition:
   ```
   IS ABOVE 80
   ```

5. Under **Evaluations**, set:
   - Evaluate every: `1m`
   - For: `1m`

6. Set **Contact point**  (Email)
 to set contact point go to **Alerting > Contact points**.

7. Save the rule.

   ```avg(rate(node_cpu_seconds_total{mode="user"}[1m])) * 100 > 80
   ```

   Bonus -  **Set Up Email Notifications in Grafana**

   1. Open Config - sudo nano /etc/grafana/grafana.ini
   2. Find smtp and configure - [smtp]
enabled = true
host = smtp.gmail.com:587
user = your-email@gmail.com
password = your-app-password
from_address = your-email@gmail.com
from_name = Grafana Alerts

For Gmail The password should be the 16 digit generated from creating an app for the email alerts in the app security section of google account





## Step 5: Security and Backups

1. **SSH Key Authentication Only**  
   - Edit `/etc/ssh/sshd_config` to restrict SSH access to key-based authentication only:  
     ```bash
     PasswordAuthentication no  
     PermitRootLogin no  
     PubkeyAuthentication yes
     ```
   - Restart SSH:  
     ```bash
     sudo systemctl restart sshd
     ```

2. **Backup Script**  
   Create a backup script and make it executable:  
   ```bash
   sudo nano /usr/local/bin/wordpress-backup.sh  
   chmod +x /usr/local/bin/wordpress-backup.sh
   ```

3. **Automate Backups**  
   Set up a cron job for automated backups:  
   ```bash
   sudo -u dokku crontab -e  
   ```

   Add the following entry to run backups daily at 2 AM:  
   ```
   0 2 * * * /usr/local/bin/wordpress-backup.sh
   ```

4. **Restore From Backup**  
   To restore, follow these steps:  
   - Restore WordPress files:  
     ```bash
     tar -xzvf /var/backups/wordpress/wordpress-app-YYYY-MM-DD.tar.gz -C /home/dokku/wordpress-app
     ```
   - Restore MySQL database:  
     ```bash
     dokku mysql:start wordpress-db  
     dokku mysql:import wordpress-db < /var/backups/wordpress/mysql-YYYY-MM-DD.sql
     ```
   - Restore environment variables:  
     ```bash
     dokku config:set wordpress-app $(cat /var/backups/wordpress/wordpress-app-env-YYYY-MM-DD.txt)
     ```


## What Works and What i would improve
1. Every step works from the local setup to app and CI/CD deployment deployment. I stressed the CPU and noted the new reading in my grafana dashboard, to confirm its working plus the conection with prometheus. The security hardening and backup are also working.

## What i would improve
1. If not for the constraints i would have chosen cloud deployment and backup instead of local.
2. Alos i would add more detailed metrics to monitor such as network traffic and mayne appl level metrics.