# Part 3: Bash Scripts - Server Preparation Guide for Web Hosting

## Bastion Host Preparation
1. **Bastion Host Deployment:**
   - The VM deployed in the hub resource group serves as the bastion host.
   - It will act as our Ansible controller.
   - We will install Jenkins on this server. Jenkins will be used as a dashboard running freestyle projects.
   - In the background, it will run Ansible playbooks against our web servers to perform different actions.
   - Jenkins with the [Active Choice Plugin](https://plugins.jenkins.io/uno-choice/) will allow us to pass parameters to our Ansible scripts.
   - We will also install [phpMyAdmin](https://www.phpmyadmin.net/) for database administration.

### Server Hardening
1. **Get Repository on the Server:**
   ```sh
   cd /tmp
   wget https://github.com/chinmaymjog/web_hosting/archive/refs/heads/main.zip
   unzip main.zip 
   ```

2. **System Update & Package Installation:**
   - Update the system, install required packages, add a timestamp to history, add a banner for SSH login, secure SSH server, and enable the Ubuntu firewall.
   ```sh
   cd web_hosting-main/scripts
   chmod +x server_hardening.sh
   ./server_hardening.sh 
   ```

3. **Create LVM on Attached Data Disk & Mount it on `/data`:**
   - Mount the attached data disk on the server with LVM.
   - Note: In the script, change the disk variable to match your attached data disk. Usually, on an Azure VM, the new disk is at `/dev/sdc` unless you reboot the system.
   ```sh
   chmod +x datadisk_lvm.sh
   ./datadisk_lvm.sh 
   ```

4. **Install Ansible & Jenkins:**
   - Refer to the official documentation for installation:
     - [Jenkins Installation](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
     - [Ansible Installation](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu)
   - I have created a script for it:
   ```sh
   chmod +x install_ansible_jenkins.sh
   ./install_ansible_jenkins.sh
   ```

5. **Copy Ansible Playbooks & Config Files:**
   ```sh
   cd ..
   sudo mv ansible/ /data/
   sudo mv jenkins-jobs/ /data/
   ```

6. **Ansible Configuration:**
   - Move Ansible config file to the default location. Make a copy of the default config file first.
   ```sh
   sudo mv /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg-org
   sudo cp ansible/ansible.cfg /etc/ansible/
   ```

   - Verify the configuration:
   ```sh
   ansible-config view
   ```

   - Update the sample host inventory with correct web server private IPs Output from production & preproduction deployment in [Part 2](./Part_2.md#deploying-web-resources):
   ```sh
   ansible prod_webservers -m ping
   ansible preprod_webservers -m ping
   ```

7. **Create Backup Directory for Jenkins:**
   - Later, install the backup plugin and set this as the backup location for periodic Jenkins full backup:
   ```sh
   sudo mkdir -p /data/jenkins-bkp
   sudo chown jenkins:jenkins jenkins-bkp/
   ```

8. **Initial Jenkins Setup:**
   - Follow [this tutorial](https://youtu.be/8fVOdFdzlKc?t=348) for the initial setup.
   - Navigate to Dashboard > Manage Jenkins > Available Plugins. Search for and install the following plugins:
     - Active Choices Plug-in
     - Thinbackup 
     - Environment Injector Plugin

9. **Set Jenkins Backup:**
   - Navigate to Dashboard > Manage Jenkins > System. Look for ThinBackup Configuration.
   - Set Backup directory to `/data/jenkins-bkp`.
   - Schedule full backups to `H 12 * * *`.
   - Save the changes.

10. **Copy Jenkins Job Definitions:**
    ```sh
    cd /data/jenkins-jobs/
    sudo cp -avr . /var/lib/jenkins/jobs/
    sudo chown -R jenkins:jenkins /var/lib/jenkins/jobs/
    systemctl restart jenkins.service 
    ```

11. **Verify Jenkins is Back Online.**

## Web Host Preparation
From the Bastion host, SSH into the web server. The Terraform deployment for production or preproduction in Part 2 outputs web server IPs.

Example:

```sh
ssh 10.0.2.4
```

Now, mount the NetApp volume on the /netappwebsites directory.

Steps:
1. Create the mount directory:

```sh
sudo mkdir -p /netappwebsites
```

2. Follow [Azure NetApp Files documentation](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-mount-unmount-volumes-for-virtual-machines#mount-nfs-volumes-on-linux-clients) to get the correct mount command for your environment.

3. Run the mount command (this example assumes an NFS mount):

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