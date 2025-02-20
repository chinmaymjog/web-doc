# Part 3: Bash Scripts - Server Preparation Guide for Web Hosting
## Introduction

[Repository link](https://github.com/chinmaymjog/web-scripts.git)

## Directory Structure
- **bash** - Contains shell scripts. 

## Topics covered
- [Bastion Host Preparation](#bastion-host-preparation)
- [Web Host Preparation](#web-host-preparation)


## Bastion Host Preparation
   - The VM deployed in the hub resource group serves as the bastion host.
   - It will act as our Ansible controller.
   - We will install Jenkins on this server. Jenkins will be used as a dashboard running freestyle projects.
   - In the background, it will run Ansible playbooks against our web servers to perform different actions.
   - Jenkins with the [Active Choice Plugin](https://plugins.jenkins.io/uno-choice/) will allow us to pass parameters to our Ansible scripts.
   - We will also install [phpMyAdmin](https://www.phpmyadmin.net/) for database administration.

1. **SSH into Bastion Host**  
   Use the Bastion VM IP & SSH key generated nn [Part 2](Part_2.md#deploying-hub-resources). 
   ```sh
   ssh -i sshkey/azureuser_rsa azureuser@4.213.89.107
   ```
   The necessary scripts are available [here](https://github.com/chinmaymjog/web-scripts.git).

2. **Clone Repository on the Server:**
   ```sh
   sudo git clone https://github.com/chinmaymjog/web-scripts.git
   cd web-scripts/bash
   ```

3. **System Update & Package Installation:**  
   Update the system, install required packages, add a timestamp to history, add a banner for SSH login, secure SSH server, and enable the Ubuntu firewall.
   ```sh
   chmod +x server_hardening.sh
   ./server_hardening.sh 
   ```

4. **Create LVM on Attached Data Disk & Mount it on `/data`:**  
   Mount the attached data disk on the server with LVM.  
   Note: In the script, change the disk variable to match your attached data disk. Usually, on an Azure VM, the new disk is at `/dev/sdc` unless you reboot the system.
   ```sh
   chmod +x datadisk_lvm.sh
   ./datadisk_lvm.sh 
   ```

5. **Install Ansible & Jenkins:**
   
   Refer to the official documentation for installation:
   - [Jenkins Installation](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
   - [Ansible Installation](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu)
   
   I have created a script for it:
   ```sh
   chmod +x install_ansible_jenkins.sh
   ./install_ansible_jenkins.sh
   ```

6. **Copy Ansible Playbooks & Config Files:**
   ```sh
   cd /data
   sudo git clone https://github.com/chinmaymjog/web-ansible.git ansible
   ```

7. **Ansible Configuration:**  
   Move Ansible config file to the default location. Make a copy of the default config file first.
   ```sh
   sudo mv /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg-org
   sudo cp ansible/ansible.cfg /etc/ansible/
   ```

8. **Verify the configuration.**
   ```sh
   ansible-config view
   ```

9. **Update the host inventory.**   
   Update `hosts` file with correct web server private IPs Output from production & preproduction deployment in [Part 2](./Part_2.md#deploying-web-resources)
   e.g.
   ```sh
   [prod_webservers]
   prod_web1 ansible_host=10.0.1.5
   prod_web2 ansible_host=10.0.1.4

   [preprod_webservers]
   preprod_web1 ansible_host=10.0.2.5
   preprod_web2 ansible_host=10.0.2.4
   ```

10. **Test if ansible controller communicates with hosts.** 

      ```sh
      ansible prod_webservers -m ping
      ansible preprod_webservers -m ping
      ```

11. **Create Backup Directory for Jenkins:**  
   Later we will install the jenkins backup plugin and set this as the backup location for periodic Jenkins full backup:
      ```sh
      sudo mkdir -p /data/jenkins-bkp
      sudo chown jenkins:jenkins /data/jenkins-bkp/
      ```

12. **Initial Jenkins Setup:**
   - Browse the bastion host IP on port 8080 to access Jenkins
   e.g. http://4.213.89.107:8080
   Follow [this tutorial](https://youtu.be/8fVOdFdzlKc?t=348) for the initial setup.
   - Navigate to Dashboard > Manage Jenkins > Available Plugins. Search for and install the following plugins:
      - Active Choices Plug-in
      - Thinbackup 
      - Environment Injector Plugin

13. **Set Jenkins Backup:**
   - Navigate to Dashboard > Manage Jenkins > System. Look for ThinBackup Configuration.
   - Set Backup directory to `/data/jenkins-bkp`.
   - Schedule full backups to `H 12 * * *`.
   - Save the changes.

14. **Copy Jenkins Job Definitions:**
      ```sh
      cd /data
      sudo git clone https://github.com/chinmaymjog/web-jenkinsjobs.git jenkinsjobs
      cd /data/jenkinsjobs/jobs
      sudo cp -avr . /var/lib/jenkins/jobs/
      sudo chown -R jenkins:jenkins /var/lib/jenkins/jobs/
      sudo systemctl restart jenkins.service 
      ```

15. **Verify Jenkins is Back Online.**
   Browse the bastion host IP on port 8080 to access Jenkins
   e.g. http://4.213.89.107:8080

## Web Host Preparation
From the Bastion host, SSH into the web server. The private key has been placed on bastion host under users home directory during terraform provisioning. Use IPs from the Terraform deployment for production or preproduction in [Part 2](./Part_2.md#deploying-web-resources)

Example:

```sh
ssh 10.0.2.4
```

Now, mount the NetApp volume on the /netappwebsites directory.
Follow [Azure NetApp Files documentation](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-mount-unmount-volumes-for-virtual-machines#mount-nfs-volumes-on-linux-clients) to get the correct mount command for your environment.

Steps:
1. **Create the mount directory.**
   ```sh
   sudo mkdir -p /netappwebsites
   ```

3. **Run the mount command (this example assumes an NFS mount)**
   ```sh
   sudo mount -t nfs 10.0.2.132:/str-pprd-inc /netappwebsites -o rw,hard,rsize=262144,wsize=262144,sec=sys,vers=4.1,tcp
   ```

4. To make the mount persistent, add an entry to /etc/fstab:
   ```sh
   echo "10.0.2.132:/str-pprd-inc /netappwebsites nfs rw,hard,rsize=262144,wsize=262144,sec=sys,vers=4.1,tcp 0 0" | sudo tee -a /etc/fstab
   ```

5. Reload the system daemon and remount:
   ```sh
   sudo systemctl daemon-reload
   sudo mount -a
   ```

6. Verify the mount:
   ```sh
   df -h
   ```

Repeat these steps for all production and preproduction web servers, ensuring that each mounts its respective NetApp volume.