step 1- Updating the VM and installig dependenciees
sudo apt update
sudo apt update -y 

INstall all the required packages. 
python, pip , venv, git, build-essential libpq-dev python-3-dev

build-essential — a meta-package that installs GCC (C compiler), make, and other build tools. 
	Some Python packages contain C extensions that need to be compiled during pip install. 
	Without this those installs fail with errors like gcc: command not found.
libpq-dev — PostgreSQL development headers and libraries. Only needed if you use psycopg2 (PostgreSQL driver for Django).
python3-dev — Python development headers (.h files). Also only needed when compiling Python packages with C extensions, particularly psycopg2.

Check if all the packages are installed sucessfully
python3 --version
pip3 --version
git --version

create a folder to clone the repo
 in home: sudo mkdir -p srv/Portfoliosite
 
 Give user permission to the folder:
 sudo chown user:user /srv/Portfoliosite
 
 clone the github repo entirely inside the portfoliosite folder as:
 git clone "your github repo linkls" .
 
 create a venv inside the portfoliosite as:
 python3 -m venv venv 
 
 check if it exists by ls the "venv" should be visible
 
 Activate the virtual environment sa:
 source venv/bin/activate
 
 Install the requirements file as:
 pip install -r requirements.txt It reads every package from requirements.txt and installs it inside the venv
 
 check if installed properly by checking django version as:
 python -m django --version
 
 Now being in the venv and correct folder create the gitignore file
and paste the info amd add whatever necessary

changes in allowed host, debug false, ans static in settings.py 
  
than migrate using:
python manage.py migrate

then createsuperuer as 
python manage.py createsuperuser

create superuser 

then copy static files to staticfils as:
python manage.py collectstatic
Error I faced: staticfiles.W004 — The directory '/srv/portfoliosite/portfolio/static' does not exist
Cause: STATICFILES_DIRS in settings.py pointed to a static folder that was never created.
Fix: Created the folder with mkdir /srv/portfoliosite/portfolio/static or set STATICFILES_DIRS = [] in settings.py if all static files live inside app folders

After that test if the site is loading currently as:
python manage.py runserver 0.0.0.0:8000
NOTE: at this step the css might not render if DEBUG is kept false as gunicorn is not in action. Just make sure the page is available
After runserver test step:

Error i faced: Homepage returned 404.
Cause: No URL pattern existed for / in urls.py. Everything was mounted under portfolio_pages/ so the root path had nothing to serve.
Fix: Changed path('portfolio_pages/', include('portfolio_pages.urls')) to path('', include('portfolio_pages.urls')) in urls.py so pages are served directly at /.

After runserver test step:

Error: Homepage returned 404.
Cause: No URL pattern existed for / in urls.py. Everything was mounted under portfolio_pages/ so the root path had nothing to serve.
Fix: Changed path('portfolio_pages/', include('portfolio_pages.urls')) to path('', include('portfolio_pages.urls')) in urls.py so pages are served directly at /.

Now, install and test gunicorn(It is a production grade python web server)::
Django's built-in dev server handles one request at a time and is not safe for production. 
Gunicorn (Green Unicorn) is a proper WSGI server that handles multiple simultaneous requests 
using worker processes. It sits between Nginx and your Django code.
 The key concept: Gunicorn speaks HTTP internally, Nginx faces the public internet.

With venv active, install gunicorn
pip install gunicorn

Add gunicorn to requirements.txt so it's tracked
pip freeze > requirements.txt

Test gunicorn manually first
# Replace 'myproject' with your actual Django project folder name i.e (the folder that contains settings.py and wsgi.py)
gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 3 (still the css might not load)


then create a systemd service for gunicorn
What this does
Right now Gunicorn only runs while you have a terminal open. When you close SSH or the VM reboots, it stops. 
A systemd service makes it run as a background process and auto-restart on failure. 
This is the Linux equivalent of setting up a background service on Windows.

Inside /etc/systemd/system/
create a folder named gunicorn.service
inside this:
[Unit]
Description=Gunicorn daemon for Django portfolio
After=network.target

[Service]
User=youruser (actual linux username)
Group=www-data
WorkingDirectory=/srv/myproject (this is a directory that containes your manage.py)
EnvironmentFile=/srv/myproject/.env (make sure this address is correct)
ExecStart=/srv/myproject/venv/bin/gunicorn \
    --access-logfile - \
    --workers 3 \
    --bind unix:/srv/myproject/gunicorn.sock \
    myproject.wsgi:application

[Install]
WantedBy=multi-user.target

Reload systemd so it sees the new file
sudo systemctl daemon-reload

Enable it (auto-start on boot)
sudo systemctl enable gunicorn

Start it right now
sudo systemctl start gunicorn

Error I faced: Failed to load environment files: No such file or directory
Cause: EnvironmentFile path in gunicorn.service was pointing to the wrong location.
Fix: Corrected the EnvironmentFile path to match the actual .env file location. Ran sudo systemctl daemon-reload && sudo systemctl restart gunicorn.


# Check if it's running
sudo systemctl status gunicorn
# Look for: Active: active (running)

IF erros occurs check for bugs as 
sudo journalctl -u gunicorn 

Now install and configure nginx as:
What this does
Nginx is a high-performance web server that does two jobs here: (1) it serves your static files 
(CSS/JS/images) directly from disk — very fast, (2) it forwards all other requests to Gunicorn via 
the Unix socket. This is called a reverse proxy. The internet sees Nginx on port 80; Nginx silently forwards to Gunicorn.

> Install Nginx and enable nginx
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx


>Create a config file for your site at
 /etc/nginx/sites-available
sudo vim myportfolio

>paste the following configuration:
server {
    listen 80;
    server_name 192.168.1.105;  # Your VM's IP address

    location = /favicon.ico { access_log off; log_not_found off; } (No logs if favion.ico inorder to save form the flood of logs
								browser keep looking for favicon after a short interval)

    location /static/ {
        root /srv/myproject; (proper location to static files)
    }

    location /media/ {
        root /srv/myproject;
    }

    location / {
        include proxy_params; (look inside this folder for things other than static and media and connect through gunicorn socket)
        proxy_pass http://unix:/srv/myproject/gunicorn.sock;
    }
}

>Enable the site by creating a symlink
sudo ln -s /etc/nginx/sites-available/myportfolio      /etc/nginx/sites-enabled/

>Remove the default Nginx page
sudo rm /etc/nginx/sites-enabled/default

> Test your config for syntax errors
sudo nginx -t
# Should output: syntax is ok  configuration file test is successful

> Reload Nginx with the new config
sudo systemctl reload nginx

>Allow Nginx through the firewall
sudo ufw allow 'Nginx Full'

# Now visit from Windows browser:
# http://192.168.1.105
# Your Django site should appear with all styling!
Error: Gunicorn started but workers immediately crashed with ModuleNotFoundError traceback ending at importlib.import_module(module).
Cause: Wrong wsgi module name in the ExecStart line of gunicorn.service. The module path did not match the actual folder structure.
Fix: Ran find /srv/portfoliosite -name "wsgi.py" to find the correct path. The correct module name was portfolio.wsgi not myproject.wsgi.
 Also verified WorkingDirectory pointed to the folder containing manage.py. Updated service file, then ran sudo systemctl daemon-reload && sudo systemctl restart gunicorn.

Also the static files didnot render properly and I modified the file location after which it rendered properly.
making gunicorn restart by itself if it crases: 
command: sudo systemctl edit gunicorn  (EDITOR=vim sudo systemctl edit gunicorn to edit with vim)
code:
		[service]
		Restart=on-failure;
		RestartSec=5s

# Make sure www-data (Nginx user) can access your project
# Add your user to the www-data group
sudo usermod -aG www-data youruser

Give read+execute permission on the project folde
sudo chmod -R 755 /srv/myproject
		
# Reload both services
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo systemctl restart nginx

# Test auto-restart by killing Gunicorn manually
sudo kill -9 $(pgrep gunicorn)

# Wait 5 seconds then check
sudo systemctl status gunicorn
# It should have restarted automatically!
