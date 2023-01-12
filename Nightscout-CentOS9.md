# Nightscout Setup on CentOS Stream 9
This is still a work in progress. NO NOT FOLLOW YET!

This guide will cover how to set up an instance of [Nightscout](https://nightscout.github.io/) on a **fresh** CentOS 9 Stream machine as well as basic server security, reverse proxying with nginx, and securing nginx with LetsEncrypt. There are plenty of Nightscout guides for Ubuntu, but nothing good for RPM distrobutions (Red Hat derived distrbutions). You may be more comfortable with Red Hat Linux, or use Foreman or Satellite to manage RHEL servers and want Nightscout to be updated with the rest of your RH servers. 

There are plenty of other Red Hat distrobutions (Fedora, Suse, Rocky, Oracle, Alma, etc) but I like CentOS Stream since it's still supported by Red Hat (unlike downstream distros like Suse, Rocky, Alma, and Oracle), but is the closest upstream distro to the offical Red Hat Enterprise Linux. 

Not to get too "Inside Baseball", but the current devolpment stream is:

Rawhide —> Fedora —> CentOS stream —> RHEL —> (Rocky, Alma, Oracle, and Suse*kinda*).

CentOS Stream seems like the best "Sweetspot" for me. If you prefer a downstream distro, Godspeed to you. 

**Note: If your host didn't give you a root account, you can skip this section and "securing your server".**
Once you're inside, you should see a line on the bottom saying:
```bash
root@<machine hostname>:~$ 
```
This is the shell you will be using for the rest of this guide.

## Securing your server
While we can't run nightscout as root, we want to start as root to update the server and create the service user. 

Install EPEL (Extra Packages Enterprise Linux) for the 'screen' application
```bash
dnf install epel-release -y
```

Now install some required packages. You may already have some of these installed but it doesn't hurt to try isntalling again. 
```bash
dnf install -y screen nano git tar bash-completion
```

To update your server, run the following command (press enter at each newline):
```bash
dnf upgrade -y
```

Disable SELinux
```bash
nano /etc/selinux/config
```
Replace `SELINUX=enforcing` with `SELINUX=disabled` or `SELINUX=permissive`.

Type `Ctrl + X` and then `y` and `enter` to save and exit the file.

Open up the http, https, and 1337 ports in the firewall.
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=1337/tcp
firewall-cmd --reload
```

Next, we'll add the new non-root user. I'm going to call it `nightscout` since that's the purpose of this server. You can use any username you want but you will need to change it whenever you see me reference the nightscout account. 
```bash
adduser nightscout
```
Give the new user a password. You will need to type in the password twice in case you have "Fat Finger Syndrome" like I do.
```bash
passwd nightscout
```
Next we'll be adding the new user to the 'wheel' user group. In Red Hat distrobutions (as CentOS is) the Wheel group allows members sudo privileges. As in "The Wheel of Trust" 
```bash
usermod -aG wheel nightscout
```
It is likely that the machine's kernel was updated. If so, you will want to restart and log back in as the root user. 
```bash
reboot
```
After the server reboots, log in as the new `nightscout` user

Once you're the new account, we should check if it has administrative privileges. Run `sudo cat /etc/shadow`, The shadow file can only be accessed by root. If you did not get an error you can move on. If you got an error then you need to check that the nightscout user is a member of the 'wheel' group.

It's best practice to not allow the root user to ssh into the machine so we need to edit the SSHd configuration file to prevent the `root` from being able to log in via ssh. 

```bash
sudo nano /etc/ssh/sshd_config
```
You can use `Ctrl + W` to search for the line that says `PermitRootLogin`, and change it to `no`. Save the file, then run:
```bash
sudo systemctl restart sshd
```

## MongoDB
### Installing
The latest stable release of MongoDB is `6.0` and in order to install this version we have to add their package repository to our system.
```bash
echo '
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
' > ~/mongodb-org-6.0.repo
```
and then...
```bash
sudo mv ~/mongodb-org-6.0.repo /etc/yum.repos.d/mongodb-org-6.0.repo
```
Afterwards, run `sudo dnf check-update` to download the metadata. Not really required but why not. Then install Mongodb. 
```sh
sudo dnf install mongodb-org -y
```
Start and enable the service (enable = service will start automatically upon reboot).
```sh
sudo systemctl enable mongod.service
sudo systemctl start mongod.service
```
Check `systemctl status mongod.service` and it should be running.

### Securing MongoDB
MongoDB is unsecure by default. We should harden it a bit by adding an administrative user and enabling authentication. First, let's access a Mongo shell by running `mongosh`. 

Let's switch over to the admin database with `use admin`, and create an admin user. Replace the `<admin password here>` with a secure password. 
```sh
db.createUser({user:'admin',pwd:'<admin password here>',roles:[{role:'userAdminAnyDatabase',db:'admin'}, "readWriteAnyDatabase"]})
```

Great, now you can exit the MongoDB client with `exit`. We're now going to enable authentication so not just anyone can read your data.

```sh
sudo nano /etc/mongod.conf
```

You should now use `Ctrl + W` to find the `security` section, which is commented out. Uncomment it by removing the pound sign (`#`) and add an `authorization` field.
```sh
security:
  authorization: enabled
```

Save the file, then restart Mongo.
```bash
sudo systemctl restart mongod
```

Now let's create a database and a user that can read/write to that database our Nightscout instance will use. First, we will connect to Mongo once again but this time we must authenticate the connection.
```bash
mongosh -u admin -p --authenticationDatabase admin
```
Enter in your password, then you should be be in the Mongo shell once again. Awesome, let's swap to a new database and create the user. Replace the `<nightscout user password here>` with a secure password. 
```bash
use nightscout
db.createUser({user:'nightscout',pwd:'<nightscout user password here>',roles:[{role:'readWrite',db:'nightscout'}]})
db.createCollection("entries")
```
Exit the Mongo shell once again with `exit`, and move onto "Setting up Nightscout",

## Setting up Nightscout
As of the writing of this guide, nightscout is not compatible with nodejs stream 18 and version 18 is the only one available using dnf. So we need to download the latest compatible version by other means. 
```bash
sudo curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
source ~/.bash_profile
nvm install v14.21.2
```
Now it's time to clone the Nightscout repository to your machine and install Nightscout.
```bash
git clone https://github.com/nightscout/cgm-remote-monitor.git
cd cgm-remote-monitor
npm install
```

We can now set up our configuration file. Run `nano my.env`, and paste the following in:
```bash
MONGODB_URI=
BASE_URL=
API_SECRET=
MONGODB_COLLECTION=entries
DISPLAY_UNITS=
ENABLE=careportal%20iob%20cob%20openaps%20pump%20bwg%20rawbg%20basal%20bridge%20delta
BRIDGE_USER_NAME=
BRIDGE_PASSWORD=
TIME_FORMAT=12
THEME=colors
CUSTOM_TITLE="My Self-Hosted Nightscout yo"
BG_TARGET_TOP=150
BG_TARGET_BOTTOM=70
BG_HIGH=250
BG_LOW=69
HOSTNAME=0.0.0.0
INSECURE_USE_HTTP=true

```

- `MONGODB_URI`: `mongodb://<night scout user>:<nightscout user password>@127.0.0.1:27017/nightscout`
- `BASE_URL`: your server IP + the port you want nightscout to run on, for example `127.0.0.1:1337`
- `API_SECRET`: select this yourself, it's supposed to be a secret!
- `MONGODB_COLLECTION`: the collection we created earlier Nightscout will be storing all its data in
- `DISPLAY_UNITS`: should be either `mg/dl` or `mmol`, whatever floats your boat. this will default to `mg/dl` if it is not specified
- `PLUGINS`: the "careportal" and "iob" plugins are chosen by default to show how to specify multiple plugins, please read below to select your own
- `BRIDGE_USER_NAME`: the username for your dexcom account. Used if you are using the Dexcom Bridge for data. 
- `BRIDGE_PASSWORD`: the password for your dexcom account. 
- `HOSTNAME`: While this variable doesn't seem to matter with Ubuntu machines, it does for CentOS. Don't forget this variable.

This is a very barebones configuration to get Nightscout up and running, in order to see a list of the values you can provide, look at the [README](https://github.com/nightscout/cgm-remote-monitor#environment).

Next, let's create a start script for Nightscout. Run `nano start.sh` and paste the following:
```bash
(eval $(cat my.env | sed 's/^/export /') && PORT=1337 node server.js)
```

Make the script executable with `chmod +x ./start.sh`, then it's finally time to try it out.

```bash
screen
./start.sh
```

`screen` is used to allow the nightscout application to continue running even after you disconnect from your SSH session. While this won't be needed after we create the SystemD node, it's useful while we test functionality and create the SSL certificates. You can detach from your `screen` session by pressing `Ctrl + A`, then `D`. You can reattach it with the `screen -x` command. Make sure to detach from your `screen` session before you exit out of your `ssh` session.

Determine the local IP address of your nightscout server by typing `ip -c a | grep inet`. The purple fonted address that is not 127.0.0.1 should be your local IP address. 

Navigate your browser to that address plus `:1337`, as in `http://192.168.1.110:1337`.

If you receive some issue about SSL in your browser, make sure that you have `INSECURE_USE_HTTP=true` in your `my.var` file. If you don't intend to set up a reverse proxy with Nginx or Apache, keep this field as it is. 

If you didn't get any errors when installing nodejs, npm, or start.sh, you should see your BG trend in nightscout!

### Reverse Proxy
Assuming you have a domain with an A record pointing towards your machine, we are now going to set up a reverse proxy for Nightscout using nginx, then encrypting the connection with LetsEncrypt. Let's start by installing nginx.
```
sudo dnf install nginx -y
```

Now let's set up a server block, use `sudo nano /etc/nginx/conf.d/nightscout.conf` and paste the following in:
```
server {
    listen 80;

    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```


Check if your configuration was proper with `sudo nginx -t`, if it says it's ok, restart nginx with `sudo systemctl restart nginx`. We should now install Certbot so we can secure Nginx with LetsEncrypt.
```bash
sudo dnf install certbot -y
sudo dnf install python3-certbot-nginx mod_ssl -y
sudo certbot --nginx -d example.com
```
It will ask for your email and to agree to their terms and conditions. Enter in the information they ask for, press "2" when it asks whether you'd like to redirect HTTP traffic to HTTPS, then LetsEncrypt will be set up.

You should now modify your Nightscout configuration to remove the `INSECURE_USE_HTTP` field if you haven't already, reattach your screen session with `screen -x`, and re-run Nightscout by hitting `Ctrl + C` to stop the current process, and hitting the "up" button on your keyboard to repeat the start command.

That's all folks!
