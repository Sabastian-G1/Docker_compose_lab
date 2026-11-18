# Docker setup
1- Took a snapshot of the Proxmox VM prior to starting

2-Ran the ssh sabastian@10.30.27.50 command to SSH into my VM to run my commands

3- to get the tools for docker's signing key i ran the command - sudo apt update && sudo apt install -y ca-certificates curl

4- Then I used the sudo install -m 0755 -d /etc/apt/keyrings command to make a folder where apt keeps the signing keys

5-got dockers signing key and let apt read it sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

6- let apt read the key sudo chmod a+r /etc/apt/keyrings/docker.asc

7- Added Dockers apt repository for my Ubuntu realease echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list

8- refreshed the package lists sudo apt update

9-Installed Docker engine and the Compose plugin sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

10- Started Docker sudo systemctl enable --now docker

11-confirmed it with systemctl is-enabled docker

12- Again sudo docker run --rm hello-world

13- anyone in docker group can use sudo - sudo usermod -aG docker $USER
	
14-confirmed compose plugin is installed - docker compose version

15- made the directory for the project with mkdir -p ~/myapp, then cd into it

16- Created compose.yaml - nano compose.yaml

17- I put this in the nano services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    ports:
      - "8080:3001"
    volumes:
      - ./data:/app/data 
	  
18- Next I validated the file with sudo docker compose up -d

19-I then checked that it was running with docker compose ps

20- I then went to the command line of my Proxmox VM and tested it from there with the command curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080

21-Went to google on my computer and typed http://10.30.27.50:8080, then set up the admin account


22-ran the sudo reboot command, waited a minute then ssh'ed back in, checked everything was working with docker compose ps

23-I made an ed 25519 key pair for the ssh keys, I ran this on a separate PowerShell tab ssh-keygen -t ed25519 -C "SysAdm-project2"

24- I then ran the super long command to get my public key over to my VM - type $env:C:\Users\Sabas\.ssh\id_ed25519.pub | ssh sabastian@10.30.27.50 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
	This took like half an hour for me to find out what to do and fix errors that I kept making
	
25- I ssh'ed back into the vm - ssh sabastian@10.30.27.50 It asked for both my password and the key passphrase

26-I found every line that has PasswordAuthentication, noting them to get rid of them later, - grep -ri PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
	/etc/ssh/sshd_config:#PasswordAuthentication yes
	/etc/ssh/sshd_config:# PasswordAuthentication.  Depending on your PAM 	configuration,
	/etc/ssh/sshd_config:# PAM authentication, then enable this but set P	asswordAuthentication
	grep: /etc/ssh/sshd_config.d/50-cloud-init.conf: Permission denied
	
I realized I ran the command without sudo, causing the permission denied so I ran sudo !! and got  /etc/ssh/sshd_config:#PasswordAuthentication yes
	/etc/ssh/sshd_config:# PasswordAuthentication.  Depending on your PAM 	configuration,
	/etc/ssh/sshd_config:# PAM authentication, then enable this but set 	PasswordAuthentication
	/etc/ssh/sshd_config.d/50-cloud-init.conf:PasswordAuthentication yes
	
27- I have to now nano into sshd_config and turn PasswordAuthentication to no - sudo nano /etc/ssh/sshd_config, then ctrl+w PasswordAuthentication and then change the yes to no. I also had to get rid of the # to make it not a comment

28- I did the same thing for the 50-cloud-init.conf file with - /etc/ssh/sshd_config and the same strategy

29- to check if everything works I ran sudo sshd -t, had no output so all good

30- Since sshd only reads its settings when it starts I had to restart ssh - sudo systemctl restart ssh

31- To check it worked, in a different tab I ran ssh -o PubkeyAuthentication=no sabastian@10.30.27.50
	I received the message Permission denied (publickey), so all good
	
32- I dropped all incoming traffic and then allowed all outgoing traffic - sudo ufw default deny incoming     sudo ufw default allow outgoing

33- To avoid locking myself out I allowed OpenSSH - sudo ufw allow OpenSSH

34 - I allowed port 8080 - sudo ufw allow 8080/tcp

35 - I then turned it on with sudo ufw enable

36 - I checked the status with sudo ufw status verbose 
	Status: active
	Logging: on (low)
	Default: deny (incoming), allow (outgoing), deny (routed)
	New profiles: skip
	To                         Action      From
	--                         ------      ----
	22/tcp (OpenSSH)           ALLOW IN    Anywhere
	8080/tcp                   ALLOW IN    Anywhere
	22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
	8080/tcp (v6)              ALLOW IN    Anywhere (v6)
	
37- To check again I ran the nmap -Pn -p- 10.30.27.50 command, got nothing and then the sudo ss -tlnp and thise was the output
	State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
	LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=677,fd=13))
	LISTEN 0      128          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=72015,fd=3))
	LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*     users:(("docker-proxy",pid=1319,fd=7))
	LISTEN 0      128             [::]:22            [::]:*     users:(("sshd",pid=72015,fd=4))
	LISTEN 0      4096            [::]:8080          [::]:*     users:(("docker-proxy",pid=1326,fd=7))
	
38- I ran these commands to set up automatic updates - sudo apt install -y unattended-upgrades, it was already the newest version so ll good, then the sudo dpkg-reconfigure -plow unattended-upgrades command 

39- To ensure that it worked i ran cat /etc/apt/apt.conf.d/20auto-upgrades, both lines produced a 1, then I ran sudo unattended-upgrade --dry-run --debug, which gave me a wall of text then said that there were no packages to be upgraded unattended

40- I made a tar backup script with nano ~/backup.sh, I then put this in the nano
	#!/usr/bin/env bash
	#stop the app, save it's data then restart it
	set -euo pipefail
     	    #keeps the newest archives
	APP_DIR=/home/sabastian/myapp
    	    #where compose.yaml is
	BACKUP_DIR=/home/sabastian/backups
       	 #where the archives go
	KEEP=7
        #how many to keep
	mkdir -p "$BACKUP_DIR"
	cd "$APP_DIR"
	docker compose stop
        	#stops the app so nothing changes whilst tar copies it
	tar -czf "$BACKUP_DIR/myapp-$(date +%Y%m%d-%H%M%S).tar.gz" -C "$APP_DIR" data
        	#archives the data folder in a timestamped .tar.gz
	docker compose start
        	#starts the app again
	ls -t "$BACKUP_DIR"/myapp-*.tar.gz | tail -n +$((KEEP + 1)) | xargs -r rm --
	
41-To make sure there are not any error I ran - bash -n ~/backup.sh, that gave me no line number back so all good, I then gave it execute permissions with chmod +x ~/backup.sh

42- I then ran it twice to make sure there is more than one archive with sudo ~/backup.sh

43- I ran ls -lh ~/backups to see the backups 
	-rw-r--r-- 1 root root 15K Nov 18 20:40 myapp-20261118-204004.tar.gz
	-rw-r--r-- 1 root root 15K Nov 18 20:40 myapp-20261118-204021.tar.gz
	
44- Now in Proxmox I went to the backups tab and pressed backup now, kept the storage as default, and changed the mode to snapshot

45- Going back to my webpage I pressed add new monitor and then put the friendly name as Sabastian 11/18, then put the URL as https://example.com, then I saved

46- I ran the backup script with sudo ~/backup.sh, I then ran ls -lh ~/backups and there was three .tar.gz files

47- I went to the myapp directory, stopped and removed the containers with sudo docker compose down

48- I then deleted the app's data with sudo rm -rf data, I then ran ls just to make sure that it was gone, it was

49- to get everything back i ran the command sudo tar -xzf ~/backups/myapp-20261118-210039.tar.gz -C ~/myapp, I followed that up by running ls and the data folder was back

50- to get the app back up again I ran sudo docker compose up -d, followed by sudo docker compose ps

51- I went back to my webpage and reloaded, as I had reloaded after I composed down docker, and everything was back to where it was supposed to be with no issues.

# LINKS
Please note that all these links are rough as I did not realize I had to record the links and had to go back to find the pages that I had used to figure out what I was doing. 






