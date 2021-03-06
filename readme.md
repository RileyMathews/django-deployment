# deploying django app to digital ocean

This is a step by step guide in how to go from absolutely no code or virtual machine to a deployed app on digital ocean that is one to one matched with your local development environment. This is not nessesarily a full production deployment guide but can be useful for getting your django project on a live ip address for testing, demos, or other purposes.

This guide will assume understanding of basic django development principals and shell commands.

## notes on what should be matching
anywhere you see here the word myproject wrapped in {} like {myproject} that should be replaced with the name of the directory you choose for your directory that is your django project which you run your 'manage.py' file from.

anywhere you see {virtualuser} replace it with the username you created for your virtual machine

anywhere you see {env} replace it with the name of your virtual environment directory name

anywhere you see {ip} replace it with the ip address for your droplet

when you see {github} replace it with the link to your github repo code

when you see words formatted like that you will not actually type out the {} brackets

## create your digital ocean droplet
1. on digital ocean create a droplet using ubuntu version 18.*
1. if you do not have a key set up with digital ocean yet you will need to add one. If you do not have any ssh key setup on your machine you will need to do that first.
1. once your droplet is ready click 'create'
1. get the ip address for your droplet. that is all we will need for the time being

## create your local django code
1. on your local machine create a directory that will serve as the parent container for your project
```
mkdir {myproject}
cd {myproject}
```
1. create a virtual environment for development `virtualenv {env}`
1. activate your virtual environment `source {env}/bin/activate`
1. install django `pip install django`
1. create your django project `django-admin startproject {myproject}`
1. cd into the directory `cd {myproject}`
1. touch a gitignore file and add anything you would for your normal django development workflow
1. create a pip requirements file with `pip freeze -l > requirements.txt`
1. create a git repo and push it up to a github repo
```
git init
git add .
git commit -m "initial commit"
git remote add origin {github}
git push origin master
```

At this point do what you need to make sure your django project can run locally and loading up the web page will present the default django landing page. If your django project will not run locally troubleshoot that until it is working before moving on. 

## setup django for deployment
1. open up the django projects settings.py file and find the line that starts with ALLOWED_HOSTS, change that line to read like this, substituting in the ip address from your digital ocean droplet `ALLOWED_HOSTS = ['{ip}', '127.0.0.1', 'localhost']`
1. if you already know what domain or subdomain your app will be living on you can also add that to the list as well, to do a catchall for sub domains you can list it as `['*.exampledomain.com']`, the * will catch any subdomain. 
1. navigate to the bottom of the file and add as the very last line `STATIC_ROOT = os.path.join(BASE_DIR, 'static')`
1. at this point confirm nothing we have done has caused any errors when running the project locally
1. if everything still works commit changes to master and push to github


## preparing your digital ocean server
Rather than copy a bunch of commands go read this wonderful guide already written about how to setup your vps. 
Go through every step until the point you have your firewall setup
https://github.com/stevebrownlee/vps-setup

## install oh my zsh (optional)
If you would like to have something slightly more user friendly than the default bash terminal follow these steps 

1. run `sudo apt install zsh`
1. then run `sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"`

If the above throws errors you likely forgot to install curl and or git while setting up your virtual machine.

## setup your virtual machine with your django code
The first thing you will need to do is generate a deploy key on your virtual machine, and add it to your github repo so you are allowed to pull in your code. 
1. while logged into your created user on your virtual machine run `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`
1. get the key by running `cat ~/.ssh/id_rsa.pub`
1. copy the output
1. go to your github repo, and click on settings
1. in the side nav click on deploy keys
1. add a new deploy key and copy in the key you copied from your virtual machine terminal
1. your virtual machine is now ready to pull in your code

Now then we will install a few extra things to run your django code

```
sudo apt update
sudo apt install python3-pip python3-dev sqlite3 curl gunicorn
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv
```

Now then decide where you want the project to live on your virtual machine. We will follow the same file structure as we did on your local machine. A virtual environment directory that will live as a sibling to your git repo

1. From the directory you have decided run `virtualenv {env}`
1. activate the environment with `source {env}/bin/activate`
1. now then pull in your github code with `git clone {github}`
1. if the clone command got a permissioned denied error you likely forgot to add your deploy key to the github project

## start your project
1. cd into the repo and run
```
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py collectstatic
sudo ufw allow 8000
python manage.py runserver 0.0.0.0:8000
```
1. now open up your browser and navigate to the ip address of your droplet followed by :8000 for example if your ip adress is 123.123.123.12, navigate to 123.123.123.12:8000
1. if everything worked correctly you should see your django landing page. 

At this point you could leave everything as it, suspend the process run it in the background and quietly leave the virtual machine with the dev server still serving your application. However this would cause some problems if you want to log back into your server and stop the dev server to update your code or do anything really. if somehow the port is accidentally left running you can kill the process by doing the following...

1. find the process that is running on port 8000 by running `sudo lsof -i:8080`
1. this should print out the process, get the id number and run `kill [process number]`
1. the django server should now be stopped

## testing gunicorn can serve the application
In your virtual machine, make sure you are in your virtual environment and cd into the root of the django project
run
```
pip install gunicorn
gunicorn --bind 0.0.0.0:8000 {myproject}.wsgi
```
Navigating to ipaddress:8000 in your browser should once again display the splash page


## create gunicorn configuration files
We will now create some configuration files that will help you in managing your gunicorn server
run `sudo nano /etc/systemd/system/gunicorn.socket`
copy in the following configuration and save the file
```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```
and save the file

next run `sudo nano /etc/systemd/system/gunicorn.service`
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User={virtualuser}
Group=www-data
WorkingDirectory=/directory/to/rootof/djangoproject
ExecStart=/directory/to/environment/env/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          {myproject}.wsgi:application

[Install]
WantedBy=multi-user.target
```
Note the WorkingDirectory and ExecStart paths above should be the path to django and environment directories on your machine, these will be different based on your app and environment names. 

test the install worked by running...
```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

if everything wen't corrently there shouldn't be any obvious errors. 
if you get errors you can start tracing the error by running either of these two commands

```
sudo systemctl status gunicorn.socket
sudo journalctl -u gunicorn.socket
```

The most likely issue is something is mispelled in the two files you just created

Once you have fixed any errors. you can completely restart the entire process by running

```
sudo systemctl stop gunicorn.socket
sudo systemctl disable gunicorn.socket
sudo systemctl daemon-reload
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

## testing that all that stuff worked
run `curl --unix-socket /run/gunicorn.sock localhost`
you should see a bunch of html printed out into your terminal

if that worked then you can move onto the next step


## setting up nginx
run `sudo nano /etc/nginx/sites-available/{myproject}`

```
server {
    listen 80;
    server_name {ip};

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /directory/of/django/project;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```
Once again note: server_name, and 'root' path above should be changed based on your machines ip and environments.
server_name can also be the domain name you are going to use for the project. 

create a systemlink to the sites-enabled directory with 
```
sudo ln -s /etc/nginx/sites-available/{myproject} /etc/nginx/sites-enabled
```
check for syntax errors with `sudo nginx -t`

restart nginx with `sudo systemctl restart nginx`

to finish serving the application run 
```
sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
```



at this point you can start developing your app locally. and when you are ready to push log back into your server and pull down the new master branch from github

you may need to restart services after this happens with 

```
sudo systemctl restart gunicorn
sudo systemctl restart nginx
```

## recommended further research
Some fun things that you can research further to improve the deployment of your project

### automated deployment
There are tools such as 'fabric' 'ansible' that might be useful in automating deployment from your local repo into the virtual machines code base. 

### switching from sqlite3 to postgresql
I used sqlite3 on my deployment server so that the development database and deployment database is the same. This is to make sure code can be seamlessly integrated and we can keep the development environment as close as possible to the production environment. sqlite3 can handle small usage, i've heard by word of mouth around 1,000 users per day. If you think your application will need more than that you may want to look at transitioning your app to use postgresql. It is recommended if you switch to also use postgresql locally while developing.

### deploying your droplet on a domain
You can quite easily register a domain and point it to the ip address of your droplet

### registering your site for secure connections
if you have a domain registered for the site and wish to secure the connections you can follow this guide at 
https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-18-04
