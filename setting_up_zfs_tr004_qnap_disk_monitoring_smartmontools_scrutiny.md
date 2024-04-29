**Prerequisites:**

* A DietPi installation on your mini PC or anything debian if you want to follow the guide closely.
* Connect your QNAP TR-004 to your homelab and ensure the dip switches are set to individual mode. For optimal ZFS performance and redundancy, it's best to start with 4 disks.
* Network connectivity between your DietPi server and the TR-004.
* Root access on your DietPi system.

**Steps**

1. **Install Base Packages and Applications:**

   ```bash
   echo ''
   echo 'Installing base packages and applications'
   echo ''
   sudo apt -t $(lsb_release -cs)-backports install -y cockpit cockpit-storaged cockpit-pcp cockpit-zfs-manager cockpit-packagekit
   ```

2. **Install ZFS:**

   ```bash
   echo ''
   echo 'Install ZFS'
   echo ''
   sudo apt update
   sudo apt -t $(lsb_release -cs)-backports install samba -y
   sudo apt -t $(lsb_release -cs)-backports install -y linux-headers-amd64 zfs-dkms zfsutils-linux
   sudo modprobe zfs
   ```

3. **Reboot Server**

   ```bash
   echo 'Rebooting the server is recommended after installing ZFS components'
   sudo reboot
   ```

4. **Create a ZFS Pool (Graphical Interface)**

   * Access Cockpit (http://<your_dietpi_ip>:9090) and log in.
   * Go to the "Storage" section.
   * Locate the "ZFS" tab.
   * Consider using RAIDZ1 for its balance of data protection and storage capacity. Choose your desired RAID configuration (understanding the trade-offs), and give your pool a name.

5. **Install Docker**

   ```bash
   echo ''
   echo 'Installing Docker'
   echo ''
   sudo apt -t $(lsb_release -cs)-backports install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
   sudo mkdir /docker 
   ```

6. **Identify Disks in Your Pools**

   ```bash
   echo 'Identifying TR-004 Disks'
   lsblk -o name,model,serial | grep TR-004 | awk '{print "/dev/"$1}'
   ```

7. **Set Up Scrutiny with Docker**

   * **Create `docker-compose.yml` in `/docker`:**

     ```yaml
     services:
       scrutiny:
         container_name: scrutiny
         image: ghcr.io/analogj/scrutiny:master-omnibus
         cap_add:
           - SYS_RAWIO
           - SYS_ADMIN
         ports:
           - "8085:8080" 
           - "8086:8086"  
         volumes:
           - ./scrutiny/config:/opt/scrutiny/config
           - ./scrutiny/influxdb:/opt/scrutiny/influxdb  
         devices:
           - /dev/sda  # Replace with your disk paths
           - /dev/sdb
           - /dev/sdc
           - /dev/sdd
           # Add more if needed 
     ```

   * **Start Scrutiny**
     ```bash
     cd /docker
     sudo docker compose up -d
     ```

8. **Configure Scrutiny**

   * **Create Directories:**
     ```bash
     cd /docker
     sudo mkdir -p scrutiny/config scrutiny/influxdb
     ```

   * **Create `collector.yaml` in `/docker/scrutiny/config`:**
     ```yaml
     version: 1

     host:
       id: ""  # Leave empty if you only have one Scrutiny instance

     devices:
       - device: /dev/sda  # Replace with your disk paths found in step 6
         type: ['jmb39x-q,0']  # Adjust as needed
       - device: /dev/sdb
         type: ['jmb39x-q,1']
       - device: /dev/sdc
         type: ['jmb39x-q,2']
       - device: /dev/sdd
         type: ['jmb39x-q,3']
       - device: /dev/nvme0  # If applicable
         type: ['auto']
     ```

   * **Restart Scrutiny:**
      ```bash
      sudo docker compose restart scrutiny
      ```
Optional: Manually Check Disk Health with smartctl
```bash
# For a detailed health check using smartctl, run the following (replacing /dev/sdb with your identified disk paths)
for i in {0..3}; do echo -; sudo smartctl -d jmb39x-q,$i -a /dev/sdb ; done
```

Use code with caution.
**Working Examples**

![image](https://github.com/michaelnyatigi/wiki/assets/28693421/2eec3d86-5edf-4061-8e81-5c262992fe9d)

![image](https://github.com/michaelnyatigi/wiki/assets/28693421/42fa672b-140b-4d3a-ab22-0308649afe49)

