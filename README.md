# Nextcloud Setup
This is how I setup my self-hosted Nextcloud server with encrypted cloud backups on the cheap.

#### Hardware
I installed Ubuntu Server 20.04 on an old Thinkpad X260 that I got for $140 on ebay a few years ago. Old laptops are great becuase you have a built in KVM and battery backup. My Nextcloud data is stored on a QNAP TS-869 Pro. It's mounted in Ubuntu via a NFS share. This is just what I had laying around.

If I could make a recommendation to those purchasing hardware, I would say to use an old laptop and then install a 1TB or 2TB Sata SSD instead of using a NAS with the NFS share. This will have better performance with good reliability. No need for RAID since you have cloud backups.

## Install and Configure Nextcloud
During the installation process for Ubuntu Server 20.04 you get the option to install Nextcloud. This is what I did. It installs the Snap version of Nextcloud. You can also do this by running <code>sudo snap install nextcloud</code>. 

Finish the installation process by navigating to <code>http://server_ip/.</code> You will be greeted by the Admin Account window. Create an admin account.

#### Move Data Directory (Optional)
As I mentioned above, I have the Nextcloud data stored on my NAS via a NFS share. To do this I had to move the Nextcloud data directory like so:
1. Enable the removable media plugin: <code>sudo snap connect nextcloud:removable-media</code>
1. Disable the snap: <code>sudo snap disable nextcloud</code>
1. Move or copy the current data directory to a new location: <code>sudo mv /var/snap/nextcloud/common/nextcloud/data /media/my/new/data</code>
1. Edit the <code>datadirectory</code> line in this config file to point to your new location. <code>sudo nano /var/snap/nextcloud/current/nextcloud/config/config.php</code>
1. Re-enable the snap: <code>sudo snap enable nextcloud</code>

#### Setup custom domain
Edit your config file at <code>/var/snap/nextcloud/current/nextcloud/config/config.php</code> and add your domain like so:
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
<code>sudo nextcloud.enable-https lets-encrypt</code>

## Backups
I run nightly backups to [Backblaze B2 cloud storage](https://www.backblaze.com/b2/cloud-storage.html). It's an AWS S3 compatable object storage that is a third of the cost of AWS S3. You get 10GB for free so you can experiment with no risk. I am backing up ~350GB for around $1.75 per month. The backups are encrypted with client-side encryption so that I can use cheap and potentially untrusted storage providors without needing to worry about privacy. 

#### Rclone
[Rclone](https://rclone.org/) is what I use to interface with Backblaze. It is compatible with many different cloud storage providers and has built in encryption. Perfect for our needs!

Install Rclone:
<code>sudo apt install rclone</code>

Configure Rclone:
<code>rclone config</code>




### Sources:
- https://www.techrepublic.com/article/how-to-install-nextcloud-with-ssl-using-snap/
- https://askubuntu.com/questions/882625/nextcloud-snap-with-data-directory-on-external-harddrive
- https://help.nextcloud.com/t/howto-add-a-new-trusted-domain/26
- https://www.linuxuprising.com/2020/05/how-to-encrypt-cloud-storage-files-with.html
- https://kevq.uk/how-to-backup-nextcloud/
