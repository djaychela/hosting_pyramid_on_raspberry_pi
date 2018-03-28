# hosting_pyramid_on_raspberry_pi
Guide to getting pyramid projects up and running using a Raspberry Pi, Nginx and uwsgi

This guide is intended to help you get your Pyramid web project up and running on a Raspberry Pi.  In this, we'll take a Pi and a blank SD card, and go through all the steps needed to get the Pi up and running with Nginx serving your Pyramid project.  A significant part of the setup is covered in Chris Warrick's excellent guide here: 

https://chriswarrick.com/blog/2016/02/10/deploying-python-web-apps-with-nginx-and-uwsgi-emperor/

- but there are some details and differences not covered there.

#Get the Pi up and running.
First step is to get the Pi up and running.  We will be using the 'Lite' Image, which means it uses a command line interface (CLI) to change the settings - no desktop environment.  This is smaller, lighter and simpler.  Download the image from here:

https://www.raspberrypi.org/downloads/raspbian/

Download the 'Raspbian Stretch Lite' image.  Then follow the instructions here:

https://www.raspberrypi.org/documentation/installation/installing-images/README.md

Use Etcher to install the image to your SD card.
<etcher in progress.png>
Once written, etcher will verify the data written to the card.
<etcher validating.png>
If all is well, you will see something similar to this.
<etcher success.png>
##Enable SSH
It's possible to use the Pi with a separate keyboard and monitor, but it's often convenient to connect to it over a network using SSH and another computer - this makes maintenance tasks easier later on.  While initial setup will be done using a monitor, this will mean access after this first phase can be done over the network. To do this, before removing the newly-created SD card, create a file called 'ssh' and put it on the boot partition of the SD card, as in step 3 here:

https://www.raspberrypi.org/documentation/remote-access/ssh/

Here's the boot partition contents before:

<boot partition ready.png>
And with the ssh file in place:
<boot partition with ssh.png>

Now it's time to boot the Pi - connect it to a monitor with a USB keyboard, and the SD card in the slot.  You should see something similar to the following:

<picture from phone>

There are a couple of things to note here - the IP address (which can be found using a network scanner or by looking at your router's DHCP client list), and once you've logged in with u:pi p:raspberry you will get a warning that the default password is still in place.  Change it now using passwd to something secure that you will remember - you'll be using it a lot!

<password change from phone>

You can now connect to your Pi via SSH.  Here, using PuTTY on a PC, with the IP address entered, there will be a warning about the server's host key. 
<putty ssh key.png>

Provided you're connected to the right device, you can trust this key, and click Yes.  You will then see a login screen, where you can enter the username pi and your new password.  You should now be logged in:

<pi logged in via ssh.png>

Now it's time to get nginx up and running.

#Installing Nginx
We'll now start installing the software we need, starting with nginx - the web server that is going to serve up our Pyramid app.  Firstly, update the installation database using:

`sudo apt-get update`

then install nginx:

`sudo apt install nginx`

You'll see a large number of packages being installed to allow nginx to run - it usually takes a minute or two.  

<installing nginx.png>

Once done, go to the IP address of your Pi in a browser:

<nginx up and running.png>

If you see this, nginx is up and running, serving its default (simple) web page.

#Installing uwsgi
uWSGI is the means by which the Pyramid app will be connected to nginx, and there are a number of elements to install.  They are covered with the following command:

`sudo apt install virtualenv python3 uwsgi uwsgi-emperor uwsgi-plugin-python3 joe git`

<uwsgi installed.png>

#Getting A Simple App Running
If this is your first time running through this process, it's a good idea to get a simple Pyramid App (with few dependencies) up and running - this makes the process of setup and fault-finding simpler as there are fewer areas to have problems.  Once this simple app is up and running, you can then configure the app you actually want to use!  If you've done this a couple of times and/or are confident of the setup so far, then you can skip the latter part of this section and move straight to getting your web app up and running.  In either case, the preliminaries (setting up the virtual environment and directories for the app) will need to be carried out.

##Configuring nginx
nginx needs to be configured to use the Pyramid app.  

'sudo joe /etc/nginx/sites-enabled/testapp.conf'

insert the following content:

server {
    # for a public HTTP server:
    listen 80;
    # for a public HTTPS server:
    # listen 443 ssl;
    server_name localhost myapp.local;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/srv/testapp/uwsgi.sock;
    }

    location /static {
        alias /srv/testapp/simple_pyramid_test/static;
    }

    location /favicon.ico {
        alias /srv/testapp/simple_pyramid_test/static/pyramid-16x16.png;
    }
}

Make sure you delete any backup files made by joe (if you are using it) - they will have ~ appended to the filename (and nginx will not be happy about this)

## Copying the project, creating the directories and virtual environment
Move to the directory where the project will be served from (/srv):
`cd /srv`
To make this process simpler (and consistent), I've put a simple pyramid app on github. First, use git to clone the test project into a 'testapp' folder:
`sudo git clone https://github.com/djaychela/simple_pyramid_test testapp`
`cd testapp`
Create a virtual environment running python 3:
`sudo virtualenv -p /usr/bin/python3 venv`
activate it using:
`source venv/bin/activate`
should change the command prompt to (venv)
running
'python'
should show that you have python3 installed (version 3.5.3 in this case below)

<virtualenv python3.png>

There can be some issues with installations not working properly on a venv that has not been deactivated, so get that out of the way to be sure:
`deactivate`
followed by
`source /srv/testapp/venv/bin/activate`

##Installing the Test Web App
To do the installation, it's important to get everything installed into the venv - and this may not be possible as it's owned by the root user.  Before solving this, I did a -lot- of head scratching and wondering if I was cut out for this sort of thing - and it's a difficult thing to search for as the behaviour isn't specific or easy to pin down.  The solution I've found is to alter the entire folder to ownership by the current user (pi), do all the installation, and then once it's all complete, change it to be owned by www-data (which nginx runs as)
sudo chown -R pi:pi /srv/testapp

Next, PasteDeploy is needed - this allows uWSGI to run the Pyramid app from the .ini file - rather than having to make changes to your application to get it to work a particular way.  In this way, you can still use the development server while you're making changes, and then upload the entire web app to your Pi server without changes.  Note that you do this as the current user because of the change in permissions made above.

`pip3 install PasteDeploy`

The next stage is to install your Pyramid Application - this is done in a similar way, but pointing to the location of the setup.py file.  Note that you point pip3 at the folder (with the trailing /), not directly at the file!

`pip3 install /srv/testapp/`

It's possible (depending on the order you do things) that the installation will fail - this can be because there is an uwsgi.sock file in the folder (this is how nginx and uwsgi communicate with each other).  If that does happen:

`sudo rm -r /srv/testapp/uwsgi.sock`

and try again!

Now, change the ownership of the entire folder to www-data with the following command:
`sudo chown -R www-data:www-data /srv/testapp`

Use the following command to run the Pyramid App via uwsgi from the command line.

`sudo /usr/local/bin/uwsgi -H /srv/testapp/venv --paste config:/srv/testapp/production.ini --socket /srv/testapp/uwsgi.sock --uid www-data --gid www-data`

You should see something similar to the below:

<uwsgi running from command line.png>

Going to your Pi's IP address in a browser should now display the Pyramid Test App:

<pyramid test app running.png>

The next step is to get uwsgi emperor working.

Stop the service for the time being:

`sudo systemctl stop uwsgi-emperor.service`

Next, the uwsgi emperor config file.  Much of the language used by the emperor is unusual (read the log files if you want to get an idea), and accordingly the apps that it oversees are referred to as vassals.  Edit/create the following file:

`sudo joe /etc/uwsgi-emperor/vassals/testapp.ini`

[uwsgi]
socket = /srv/testapp/uwsgi.sock
chmod-socket = 775
chdir = /srv/testapp
master = true
binary-path = /usr/local/bin/uwsgi
virtualenv = /srv/testapp/venv
uid = www-data
gid = www-data
processes = 1
threads = 1
plugins = python3, logfile
logger = file:/srv/testapp/uwsgi.log
paste = config:/srv/testapp/production.ini

Once this is done, re-start the emperor:

`sudo systemctl start uwsgi-emperor.service`

You should now see the same test app as before, but most importantly, it's now running under uwsgi-emperor so should work once the Pi has booted up. To check this, do a reboot with:

`sudo init 6` 

Note that you'll be disconnected from your SSH session, so you'll have to log back in to make further changes:

<init 6 result.png>

But as soon as the Pi is up and running (and without logging in), the site should be up and running.

## Installing a real app.
Now it's time to install your real app.  This is the same process as before, but you can probably forego the manual run of uwsgi.  It usually pays to halt the uwsgi-emperor service while you're making changes to the site (to avoid the .sock file being generated and stopping you installing your app).

You will follow much the same process as for the test app, but you will need to install the requirements for the app before installing the module.  This is usually done by running the following command from within your app folder:

`pip3 install -r requirements.txt`

After this, you should be able to install the app, and run it in the same way you did before - you'll need to make appropriate changes to the uwsgi and nginx config files, but those changes should be fairly simple and easy to follow when changing them from the test app settings.


