# Deploy flask app on AWS EC2 Ubuntu 22.04 using Nginx and gunicorn

In this document, we will see how to deploy a Flask application using the Gunicorn WSGI server and nginx as a reverse proxy and static files server.

We will configure Nginx to act as a front-end reverse proxy in an Ubuntu VPS. Although this guide uses Ubuntu 22.04, it was last tested on Ubuntu 23.04, Ubuntu 22.04 LTS, Ubuntu 20.04 LTS, Ubuntu 18.04 LTS, and Ubuntu 16.04 LTS in 2023 and should work on future versions of Ubuntu as well.

## Prerequisites
Before starting this guide, you should have:

A Virtual Private Server (VPS) with Ubuntu installed
A domain name configured to point to your server (optional).

Here you may use __Git Bash__ or __PuTTY__ in order to access SSH.
And for file transfer to server, we can use __FileZilla__.

Follow the steps below:

## Step 1 - Install the required packages
Login to your Ubuntu machine using SSH.

To access an EC2 instance using SSH from Git Bash, you can use the ssh command in the Git Bash terminal. Make sure you have the private key (.pem file) associated with your EC2 instance. Here's an example command:

```bash
ssh -i /path/to/your/key.pem ec2-user@your-ec2-instance-ip
```
Or if you go to connect tab in your EC2, you will find SSH connection there you can also get a guide to connect SSH.


```bash
sudo apt-get update
```
```bash
sudo apt install python3-pip python3-dev nginx -y
```
```bash
sudo apt-get install python3-venv
```

If you are using __MySQL Client__ in your python project so we'll have to install the below packages as well,

```bash
sudo apt-get install default-libmysqlclient-dev build-essential
```

You might get a prompt asking "Which services should be restarted?". Simply press the ```<spacebar>``` key to select the first option and press enter to continue the installation

## Step 2 - Create a directory (for our Flask app) and setup a virtual environment

I am deploying my app inside /home/ubuntu directory. If you want to deploy your app to a different directory, switch to that directory. If you want to check which directory you are in, fire the command below:
```bash
pwd
```

Let's now create a directory to host our Flask application
```bash
mkdir FlaskAppProject && cd FlaskAppProject
```

Create the virtual environment
```bash
python3 -m venv venv
```

Activate the virtual environment
```bash
source venv/bin/activate
```

Let's now install Flask, MySQL driver and Gunicorn using pip
```bash
pip install mysql-connector-python
pip install flask gunicorn
```

OR if you have requirement.txt file what a specific versions of packages that can be installed also,
```bash
pip install -r requirements.txt
```


## Step 3 - Creating a sample project and WSGI entry point
et us now create a sample project by entering the command below:
```bash
sudo vim app.py
```

Paste the contents below to the app.py file

```bash
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
	return 'Hello World!'

if __name__ == "__main__":
	app.run()
```

Next, we’ll create a file that will serve as the entry point for our application. This will tell our Gunicorn server how to interact with the application.
```bash
sudo vim wsgi.py
```

Next, copy the contents below to file wsgi.py
```bash
from app import app

if __name__ == "__main__":
    app.run()
```

Run Gunicorn WSGI server to serve the Flask Application When you “run” flask, you are actually running Werkzeug’s development WSGI server, which forward requests from a web server. Since Werkzeug is only for development, we have to use Gunicorn, which is a production-ready WSGI server, to serve our application.

We can test Gunicorn's ability to serve our project by running the command below:
```bash
gunicorn --bind 0.0.0.0:8000 wsgi:app
```

Folder structure so far:
```
FlaskAppProject
  |____ app.py
  |____ wsgi.py
  |____ env
```

Let's deactivate our virtual environment now:
```bash
deactivate
```

## Step 4 - Creating a systemd service
Use systemd to manage Gunicorn Systemd is a boot manager for Linux. We are using it to restart gunicorn if the EC2 restarts or reboots for some reason. We create a .service file in the /etc/systemd/system folder, and specify what would happen to gunicorn when the system reboots. We will be adding 3 parts to systemd Unit file — Unit, Service, Install

Unit — This section is for description about the project and some dependencies Service — To specify user/group we want to run this service after. Also some information about the executables and the commands. Install — tells systemd at which moment during boot process this service should start. With that said, create an unit file in the /etc/systemd/system directory

Let's now create a systemd service using the following commands:
```bash
sudo nano /etc/systemd/system/app.service
```

Now paste the contents below to this file:
```bash
[Unit]
Description=Gunicorn instance for a simple hello world app
After=network.target
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/FlaskAppProject
Environment="PATH=/home/ubuntu/FlaskAppProject/env/bin"
ExecStart=/home/ubuntu/FlaskAppProject/venv/bin/gunicorn --workers 3 --bind unix:app.sock -m 007 wsgi:app
Restart=always
[Install]
WantedBy=multi-user.target
```

Then activate and enable the service:
```bash
sudo systemctl daemon-reload
sudo systemctl start app
sudo systemctl enable app
```

A file named app.sock will be automatically created. Folder structure so far:
```
FlaskAppProject
  |____ app.py
  |____ wsgi.py
  |____ env
  |____ app.sock
```


## Step 5 - Configuring Nginx
Run Nginx Webserver to accept and route request to Gunicorn Finally, we set up Nginx as a reverse-proxy to accept the requests from the user and route it to gunicorn.

Install Nginx
```bash
sudo apt-get install nginx
```

Start the Nginx service and go to the Public IP address of your EC2 on the browser to see the default nginx landing page
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### To create a new configuration file:
You can copy the default configuration and modify it as needed:
```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/app
```

Edit the app file to suit your requirements:
```bash
sudo nano /etc/nginx/sites-available/app
```

Edit few of the things in this file 
```bash
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /home/ubuntu/FlaskAppProject;

	server_name _;

    #Or if you have IP or domain name can specify like below
    #server_name 194.195.122.189; OR server_name example.org www.example.org;

	location / {
		include proxy_params;
		proxy_pass http://unix:/home/ubuntu/FlaskAppProject/app.sock;
		#try_files $uri $uri/ =404;
	}

    # To serve the static contents

	location /static/  {
		include  /etc/nginx/mime.types;
		root /home/ubuntu/FlaskAppProject/;
	}
}
```
The location /static part of this file takes care of serving the static files through nginx.


After making the necessary changes, remove the default file at /etc/nginx/sites-enabled/ and create a symbolic link again:
```bash
sudo rm default
```
```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
```

#### Test Nginx Configuration
Always test your Nginx configuration after making changes:
```bash
sudo nginx -t
```

#### Restart Nginx
If the configuration test was successful, restart Nginx to apply changes:
```bash
sudo systemctl restart nginx
```

#### Verify the Status of Nginx
Check if Nginx is now running without issues:
```bash
sudo systemctl status nginx
```

## Step 6 - Providing necessary Permissions
Lets run the following commands to setup required permissions
```bash
sudo chmod 775 -R /home/ubuntu/FlaskAppProject
sudo chmod 775 -R /home/ubuntu
```

Restart App service and Nginx and your website should work fine!
```bash
sudo systemctl restart app
sudo systemctl restart nginx
```

On every changes in your project file, you have to reload/restart your nginx to see the changes.
```bash
sudo systemctl reload nginx
```

Tada! Our application is up!

Visit your server IP address or domain on your browser. I am visiting http://194.195.122.189/ on my browser. (note http://)
