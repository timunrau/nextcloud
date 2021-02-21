# Nextcloud Setup
This is how I setup my self-hosted Nextcloud server with encrypted cloud backups on the cheap.

#### Hardware
I installed Ubuntu Server 20.04 on an old Thinkpad X260 that I got for $140 on ebay a few years ago. Old laptops are great becuase you have a built in KVM and battery backup. My Nextcloud data is stored on a QNAP TS-869 Pro. It's mounted in Ubuntu via a NFS share. This is just what I had laying around.

If I could make a recommendation to those purchasing hardware, I would say to use an old laptop and then install a 1TB or 2TB Sata SSD instead of using a NAS with the NFS share. This will have better performance with good reliability. No need for RAID since you have cloud backups.

## Install and Configure Nextcloud
During the installation process for Ubuntu Server 20.04 you get the option to install Nextcloud. This is what I did. It installs the Snap version of Nextcloud. You can also do this by running 
```
sudo snap install nextcloud
```

Finish the installation process by navigating to <code>http://server_ip/.</code> You will be greeted by the Admin Account window. Create an admin account.

#### Move Data Directory (Optional)
As I mentioned above, I have the Nextcloud data stored on my NAS via a NFS share. To do this I had to move the Nextcloud data directory like so:

Enable the removable media plugin:
```
sudo snap connect nextcloud:removable-media
```
Disable the snap: 
```
sudo snap disable nextcloud
```
Move or copy the current data directory to a new location:
```
sudo mv /var/snap/nextcloud/common/nextcloud/data /media/my/new/data
```
Edit the <code>datadirectory</code> line in this config file to point to your new location. 
```
sudo nano /var/snap/nextcloud/current/nextcloud/config/config.php
```
Re-enable the snap: 
```
sudo snap enable nextcloud
```

#### Setup custom domain
If you're going to use a custom domain and expose this server to the internet, you'll need to edit your config file at <code>/var/snap/nextcloud/current/nextcloud/config/config.php</code> and add your domain like so:
```
  'trusted_domains' =>
  array (
    0 => '192.168.0.29',
    1 => 'cloud.example.com',
  ),
```
Setup your DNS to point to your server IP. If you are self-hosting, you will need to point it to your public IP and portforward in your router to your server. I'll leave this for you to figure out.

#### Setup Let's Encrypt
If you are using a domain you can setup Let's Encrypt with 1 command:
```
sudo nextcloud.enable-https lets-encrypt
```

## Backups
I run nightly backups to [Backblaze B2 cloud storage](https://www.backblaze.com/b2/cloud-storage.html). It's an AWS S3 compatable object storage that is a third of the cost of AWS S3. You get 10GB for free so you can experiment with no risk. I am backing up ~350GB for around $1.75 per month. The backups are encrypted with client-side encryption so that I can use cheap and potentially untrusted storage providors without needing to worry about privacy. 

#### Backblaze B2 
If you want to use this, go setup an account here: https://www.backblaze.com/b2/sign-up.html. Once your account is setup you're going to want to go create a bucket for your nextcloud backups. You'll also need to create an application key to access the bucket with Rclone. 

#### Rclone
[Rclone](https://rclone.org/) is what I use to interface with Backblaze. It is compatible with many different cloud storage providers and has built in encryption. Perfect for our needs!

Getting started:
```
sudo apt install rclone
rclone config
```
Since Rclone is going to be at the center of your backup solution, you'll want to familiarize yourself with how it works so that you will be able to do disaster recovery if ever needed. Here's a good article to get you started: https://rclone.org/b2/. Here's a good article about how the encryption works: https://www.linuxuprising.com/2020/05/how-to-encrypt-cloud-storage-files-with.html. Something I like about Rclone is that you can easily decrypt your files by setting up rclone on another machine and using the same string when setting up the crypt storage type. Be sure to save the encryption string somewhere safe like your password manager.

You will need to setup an encrypted remote that you can sync your backups to. Figure out how to use Rclone from the articles above and do that. You'll note that I am not listing any commands here. That's on purpose. You should know how to use Rclone for yourself so that you can recover your backups if ever need be. The articles I listed above are good references. **Whatever you do, do NOT use your Nextcloud server in production without having first learned how and tested recovering your backed up files on another machine. Backups are only useful if you can recover from them.**

For the rest of the tutorial I will assume your Backblaze remote is called <code>b2-encrypted</code>.

### Automated Backups
**Note:** If you didn't move your data repository you can just backup the <code>/var/snap/nextcloud/</code> directory. If you moved your data directory you will need to backup that location as well. 

#### Create a Backup User
The guide that I followed used a separate user to run the backup script for security. I just copied his thing and it worked well. If are are lazy you could just run this on root's cron or something.

```
# Create new user
sudo adduser ncbackup

# Switch to new user account
su - ncbackup

# Make directories for Logs
mkdir Logs

# Logout to switch back to normal user
logout

```

#### Setup Rclone as the backup user
Since we will run the script as the <code>ncbackup</code> user, we will need to setup rclone for him as well. If you already created a Rclone config as your normal user, you can just copy the config to the <code>ncbackup</code> user like so:
```
sudo cp ~/.config/rclone/rclone.conf /home/ncbackup/.config/rclone/rclone.conf
```

#### Backup Script
Create your backup script: 
```
nano /usr/sbin/ncbackup.sh
```
Apparently it's good practice to put scripts in that directory. 
  
Here is a starting point for your script. Modify it to suit your needs.
```
#!/bin/bash
# Output to a logfile
exec &> /home/ncbackups/Logs/"$(date '+%Y-%m-%d').txt"

# Enable Nextcloud maintenance mode before copying files
echo "enabling maintenance mode"
nextcloud.occ maintenance:mode --on

# Copy files to backblaze using rclone
echo "rclone nextcloud data to backblaze"
rclone sync -v /var/snap/nextcloud/ b2-encrypted:

echo "disable maintenance mode"
nextcloud.occ maintenance:mode --off

echo "purging old logs"
find /<path to logs folder> -mtime +14 -type f -delete

echo "Nextcloud backup completed successfully."
```
Couple of notes about the script:
- We added logging to the script. Keep an eye on these logs to ensure it's working. 
- We are using the <code>rclone sync</code> command which will just make a mirror of your files. This won't protect you from ransomware. I think Nextcloud has an app to protect against ransomware. You may want to look into that. I personally am using the recycle bin on my QNAP to protect against ransomware.

Make the script executable:
```
sudo chmod +x /usr/sbin/ncbackup.sh
```
Make it so that <code>ncbackup</code> can run the script as sudo without typing in the password:
```
# Open visudo
sudo visudo

# Allow ncbackup to run script as sudo
ncbackup ALL=(ALL) NOPASSWD: /usr/sbin/ncbackup.sh
```

#### Setup Cron
Edit crontab for the <code>ncbackup</code> user:
```
sudo crontab -u ncbackup -e
```
Once you’re in crontab, you need to add the following lines to the bottom of the file:
```
# Nextcloud backup cron (runs as 2am daily)
0 2 * * * sudo /usr/sbin/ncbackup.sh
```
#### Initial Run
You'll proabably want to manually run your script for the first time to ensure it's working as expected.
```
su - ncbackup
cd /usr/sbin
./ncbackup.sh
```
You can keep an eye on how it's doing by using tail to monitor the log:
```
tail -f /home/ncbackup/Logs/<logfilename.txt>
```
#### Bonus points - disable login on the backup user
If you want to make things really secure, you can make it so that the <code>ncbackup</code> user can't login at all and thus preventing anyone from being able to edit the backup script. 
```
sudo usermod -s /sbin/nologin ncbackup
```
If you ever need to be able to login as <code>ncbackup</code> again, you can reverse this like so:
```
sudo usermod -s /bin/bash ncbackup
```
# Sources:
- https://www.techrepublic.com/article/how-to-install-nextcloud-with-ssl-using-snap/
- https://askubuntu.com/questions/882625/nextcloud-snap-with-data-directory-on-external-harddrive
- https://help.nextcloud.com/t/howto-add-a-new-trusted-domain/26
- https://rclone.org/b2/
- https://www.linuxuprising.com/2020/05/how-to-encrypt-cloud-storage-files-with.html
- https://kevq.uk/how-to-backup-nextcloud/
