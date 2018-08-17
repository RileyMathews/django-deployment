# deploying django app through digital ocean
Welcome. This guide is a result of my frustration finding a clear concise walkthrough of how to go from a simple django project running on my local dev server. To that same code running live on the web. It is still a work in progress and I have not yet sucsessfully deployed an app. As I do my research i'm adding notes here of steps and walkthroughs i've used. My goal is eventually to have all steps layed out in this readme. This guide will not be a guide on how to sucsesfully deploy your django app in a secure, heavy traffic ready, production environment. It is simply a guide to getting a project off your local dev server and out on a live ip address to show the world. Some steps here may look like we are fully pushing a project to production such as firewall setup, and using 'production' web server tools. This is simply because these are the steps I have found that worked for me.

## setup virtual machine
1. follow all instructions at https://github.com/stevebrownlee/vps-setup for setting up your vps. Get up to the point where you have your firewall setup.
1. IMPORTANT make sure your machine is using an up to date version of python. If your running mac you might be used to simply running python, on one attempt with a vpc I really messed some things up because I did most of my setup using 'python' and 'pip install' when I should have been using 'python3' and 'pip3 install'
Make sure you have...
- a sudo enabled non root user
- nginx installed
- firewall permissions set

## setup virtual env
1. run `sudo apt-get install python-virtualenv`
1. run `sudo virtualenv /opt/myenv` This creates a virtual environment in the directory specified. 
1. run `source /opt/myenv/bin/activate` after running your command prompt should indicate the virtual environment.
1. run `pip3 install django`
1. if installing django throws a permission denied error try running `sudo chown -R [youruser]:[youruser] /opt/myenv` and then running the pip3 install again.
1. you can confirm django was installed by running `pip3 freeze -l`
1. at this point you can deactivate your virtual env by running `deactivate` 

## install PostgreSQL
We will now install postgress sql.
1. run `sudo apt-get install libpq-dev python-dev`
1. run `sudo apt-get install postgresql postgresql-contrib` These two commands install postgresql and the dependencies to make it play well with django.
1. at this point make sure you have nginx installed. if not run `sudo apt-get install nginx`

## install Gunicorn
If you have deactivated your virtual environment now is the time to get back in by running `source /opt/myenv/bin/activate`
1. run `pip3 install gunicorn`


## configure postgresql
These steps will setup your postgresql database
1. run `sudo su - postgres` your terminal prompt should change and the user should now be 'postgres'
1. run `createdb mydb`
1. create a postgres user by running `createuser --interactive -P` the interactive will prompt you for information and the -P will insist you create a password
1. run `psql` to drop into the postgres shell
1. run `GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;` passing in the user you created earlier instead of 'myuser' the ';' at the end of the line is required. you should see GRANT outputed in the terminal to confirm it worked correctly
1. from the postgress shell you can hit control+d to exit the shell. use control+d again once more to exit the postgres user and return to your original user

## get your django code
Instructions here for cloning in your git repo and editing nessesary steps to make it run in this environment

1. decide how you want to get your django code. The method here will be using git to clone in our github repo. But ultimately all of this is trying to get your django directory living under your environment directory so that /opt/myenv/(django project name here) is the root directory of your django project.
1. make sure git is installed `sudo apt-get install git`
1. you will need to setup a deploy key to pull in your code from github
1. run `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`
1. get the key you just created by running `cat /home/[username here]/.ssh/id_rsa.pub`
1. copy the key that is output into the terminal and go to the settings page of your github repo.
1. click on the 'deploy keys' tab on the sidenav and add the key.
1. you should now be able to clone in your repo into the directory 
1. `cd /opt/myenv`
1. `git clone githubprojecturl`
1. to be clear about file structure, the project you just cloned down should be a sibling to the /bin/ directory that lives under the directory that was created when your environment was created. 

Now we need to change a few settings on your project to make it run with postgres database
1. navigate to your django settings.py file and open it up in your prefered terminal text editor.
1. edit the database section of the file to look similar to...
```
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2', # Add 'postgresql_psycopg2', 'mysql', 'sqlite3' or 'oracle'.
            'NAME': 'mydb',                      # Or path to database file if using sqlite3.
            # The following settings are not used with sqlite3:
            'USER': 'myuser',
            'PASSWORD': 'password',
            'HOST': 'localhost',                      # Empty for localhost through domain sockets or           '127.0.0.1' for localhost through TCP.
            'PORT': '',                      # Set to empty string for default.
        }
    }
```
1. run pip3 install psycopg2
1. cd up one level so you are back in the root directory that holds your manage.py file
1. if your app requires database migrations do that now

## setup and start gunicorn 
Instructions here for setting up gunicorn to serve our django project

## setup and start nginx
Instructions here for setting up nginx for serving up nessesary files. 