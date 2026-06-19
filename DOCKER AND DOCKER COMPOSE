STEP 1: Install Docker and Docker Compose on the VM

  	Getting the container runtime onto your Ubuntu server
  	What this does
  	Docker is the engine that runs containers. Docker Compose is a tool that lets you define and run multiple 
  	containers together using a single YAML file. On Ubuntu you install Docker Engine, not Docker Desktop,
  	 which is only for Windows/Mac. Docker Compose v2 is now a plugin built into Docker itself, not a separate binary
  	 like it used to be.
  	 
  	First disable previous services running such as nginx and gunicorn
  	systemctl stop nginx gunicorn
  	systemctl disable nginx gunicorn
  
  	check status if disabled properly
  
  	Now install docker based on docker documentation at: http://docs.docker.com/engine/install/ubuntu/
  
  	check by the command docker run "hello world"
  
  	add your user to docker group so that you dont have to sudo each time you run a docker command as:
  	
  	sudo usermod -aG docker $USER (replace $USER with your username)
  
  	exit and again ssh to make this action to take effect
  
  	than ckeck by the command "docker ps" which should show an empty table not a permission error.
  
  	Important commands to know before starting
  
  		To build an image after the dockerfile is written (An image is a read-only snapshot of an OS+your code + dependences)
  		docker build -t myapp:latest .
  
  		A container is a running instance of that image. To run an image: 
  		docker run myapp:latest
  
  		List running containers:
  		docker ps
  
  		List all containers including stopped ones:
  		docker ps -a
  
  		List all images on your system:
  		docker images
  
  		Remove a container:
  		docker rm container_id
  
  		Remove an image:
  		docker rmi image_id
  		
  		Create a named vomule:(A volume is a folder that exists outside of all container  so data persists even if you delete and recreate containers. 
  								Without a volume, deleting a container deletes all its data permanently.)
  		docker volume create volume_name
  		
  		list volume:
  		docker volume ls
  		
  		to inspect where a volume lives on disk:
  		docker volume inspect docker_name
  		
STEP 2: Update settings.py for PostgreSQL and Docker

  	Switch database, fix decouple, standardise env vars 
  	
  	What this does
  	The settings.py needs  changes for Docker. the main one for me is:
  	switch DATABASES from SQLite to PostgreSQL. And standardise all os.getenv calls to config() so everything reads from .env consistently through one library.
  	On Windows VS Code 
  	# Open portfolio/settings.py in VS Code
  	# Make these exact changes:
  
  	#  TOP OF FILE replace the imports section with:
  	from pathlib import Path
  	import os
  	from decouple import config, Csv
  
  	BASE_DIR = Path(__file__).resolve().parent.parent
  	# REMOVED: from dotenv import load_dotenv
  	# REMOVED: load_dotenv(BASE_DIR / '.env')
  	# Reason: python-decouple reads .env automatically
  	# Having both caused double-loading conflicts
  
  	#  DEBUG  change from hardcoded True to:
  	DEBUG = config('DEBUG', default=False, cast=bool)
  	# cast=bool converts the string 'False' from .env to Python False
  	# default=False means if DEBUG is missing from .env, use False
  
  	#  DATABASES  replace the entire section:
  	DATABASES = {
  		'default': {
  			'ENGINE': 'django.db.backends.postgresql',
  			'NAME': config('DB_NAME', default='portfolio_db'),
  			'USER': config('DB_USER', default='portfolio_user'),
  			'PASSWORD': config('DB_PASSWORD'),
  			'HOST': config('DB_HOST', default='db'),
  			# 'db' is the service name in docker-compose.yml
  			# Docker's internal DNS resolves 'db' to the postgres
  			# container IP automatically — no hardcoded IPs needed
  			'PORT': config('DB_PORT', default='5432'),
  		}
  	}
  	#  EMAIL AND SOCIAL  replace with standardised config():
  	EMAIL_BACKEND = config('EMAIL_BACKEND',
  		default='django.core.mail.backends.console.EmailBackend')
  	EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
  	EMAIL_PORT = config('EMAIL_PORT', default=587, cast=int)
  	EMAIL_USE_TLS = True
  	EMAIL_HOST_USER = config('EMAIL', default='')
  	EMAIL_HOST_PASSWORD = config('EMAIL_PASSWORD', default='')
  	DEFAULT_FROM_EMAIL = config('EMAIL', default='')
  	ADMIN_EMAIL = config('EMAIL', default='')
  
  	LINKEDIN_URL = config('LINKEDIN_URL', default='')
  	FACEBOOK_URL = config('FACEBOOK_URL', default='')
  	INSTAGRAM_URL = config('INSTAGRAM_URL', default='')
  	EMAIL = config('EMAIL', default='')
  	GITHUB_URL = config('GITHUB_URL', default='')
  	# default='' means Django won't crash if a var is missing
  
  	# Pushed settings.py
  	git add portfolio/settings.py
  	git commit -m "Switch to PostgreSQL, fix decouple, standardise env vars"
  	git push origin main
  
  	On VM — SSH terminal
  	Pull the updated settings
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Test settings.py loads without import errors
  	# This runs Django's setup without starting the server
  	source venv/bin/activate
  
  	python -c "
  	import django
  	import os
  	os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'portfolio.settings')
  	django.setup()
  	print('settings.py loaded OK')
  	"
  	# Expected output: settings.py loaded OK
  	# If you see an error it will tell you exactly which line failed
  
  	=====>>  Faced an error in this phase as venv was not configured properly in the server, corrected and standarized it and it worked well.
  

STEP 3: Create Dockerfile

  	What this does
  	The Dockerfile tells Docker how to build your application image. It starts from python:3.12-slim (a minimal Python install), 
  	installs system dependencies for psycopg2, copies the code, and installs Python packages. The order matters — requirements.txt is 
  	copied before your code so Docker can cache the pip install layer and skip it on rebuilds when only your code changes.
  
  	On VS Code terminal
  	# Create a new file called 'Dockerfile' in the project root (same folder as manage.py)
  	# Capital D, no file extension
  
  	FROM python:3.12-slim
  	# python:3.12-slim = official Python image based on Debian slim
  	# slim removes many system tools keeping the image small (~130MB vs ~900MB full)
  
  	ENV PYTHONDONTWRITEBYTECODE=1
  	# Stops Python writing .pyc bytecode files — useless in containers
  
  	ENV PYTHONUNBUFFERED=1
  	# Forces Python to output logs immediately instead of buffering
  	# Without this: docker compose logs shows nothing until buffer fills
  
  	WORKDIR /app
  	# All subsequent commands run from /app inside the container
  	# Your code will live at /app inside the container
  
  	RUN apt-get update && apt-get install -y     libpq-dev     gcc     && rm -rf /var/lib/apt/lists/*
  	# libpq-dev = PostgreSQL client headers — needed to compile psycopg2
  	# gcc = C compiler — needed to compile psycopg2
  	# rm -rf /var/lib/apt/lists/* = delete apt cache to shrink image size
  
  	COPY requirements.txt .
  	# Copy requirements FIRST before any other code
  	# Docker caches each line as a layer
  	# If requirements.txt hasn't changed, Docker skips pip install entirely
  	# This saves 2-3 minutes on every rebuild when you only change Python code
  
  	RUN pip install --upgrade pip && pip install -r requirements.txt
  	# Install all Python dependencies into the container
  
  	COPY . .
  	# Copy your entire project code into /app
  	# Happens AFTER pip install so code changes don't bust the pip cache
  
  	EXPOSE 8000
  	# Documents that the container listens on port 8000
  	# Does not actually open the port — docker-compose handles that
  
  	ENTRYPOINT ["sh", "entrypoint.sh"]
  	# Run entrypoint.sh when the container starts
  	# entrypoint.sh runs migrate, collectstatic, then starts gunicorn
  
  	# Push Dockerfile
  	git add Dockerfile
  	git commit -m "Add Dockerfile"
  	git push origin main
  
  	On VM — SSH terminal
  	# Pull and do a test build of the image alone
  	# This isolates Dockerfile errors from docker-compose errors
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Build the image and tag it as portfolio-test
  	# The dot at the end means: use Dockerfile in current directory
  	docker build -t portfolio-test .
  
  	# Watch the output — each step should say DONE or CACHED
  	# The build will fail at ENTRYPOINT because entrypoint.sh
  	# does not exist yet — that is expected and OK at this stage
  	# What you are verifying is that all RUN steps succeed
  	# Check the image was created
  
  	docker images
  	# Should show portfolio-test in the list with a size around 300-400MB
  
  	# Clean up test image
  	docker rmi portfolio-test

STEP 4: Create entrypoint.sh

  	Commands that run every container startup 
  
  	What this does
  	entrypoint.sh runs three things every time the web container starts: migrate (creates/updates database tables), 
  	collectstatic (copies CSS/JS to the shared volume), then starts Gunicorn. Migrations cannot run at build time 
  	because the database container does not exist during the build. The entrypoint runs at runtime when all containers
  	are up. CRITICAL: this file must have Unix line endings (LF not CRLF) or it will fail on Linux.
  
  	# Create entrypoint.sh in your project root (same folder as manage.py)
  
  	# BEFORE SAVING: look at the bottom right corner of VS Code
  	# It shows CRLF or LF — click it and change to LF
  	# This is critical — Windows uses CRLF, Linux needs LF
  	# If you save with CRLF the script fails with 'exec format error'
  
  	#!/bin/sh
  
  	set -e
  	# set -e = exit immediately if ANY command fails
  	# Without this: if migrate fails, gunicorn starts anyway on a broken DB
  
  	echo "=== Waiting for database to be ready ==="
  	# Small sleep to let postgres fully initialise on first run
  	sleep 2
  
  	echo "=== Running database migrations ==="
  	python manage.py migrate --noinput
  	# Creates all database tables from your migration files
  	# --noinput = don't ask for confirmation
  
  	echo "=== Collecting static files ==="
  	python manage.py collectstatic --noinput
  	# Copies all CSS/JS/images from portfolio_pages/static/
  	# into /app/staticfiles/ which is the shared Docker volume
  	# Nginx reads from this volume to serve static files
  
  	echo "=== Starting Gunicorn ==="
  	exec gunicorn portfolio.wsgi:application     --bind 0.0.0.0:8000     --workers 3     --access-logfile -     --error-logfile -
  	# exec replaces the shell process with gunicorn
  	# This means docker stop sends SIGTERM directly to gunicorn
  	# --bind 0.0.0.0:8000 = listen on all interfaces port 8000
  	# --workers 3 = 3 parallel worker processes
  	# --access-logfile - = send access logs to stdout (visible in docker logs)
  	# --error-logfile - = send error logs to stderr (visible in docker logs)
  
  	# Push entrypoint.sh
  	git add entrypoint.sh
  	git commit -m "Add entrypoint script with LF line endings"
  	git push origin main
  
  	On VM — SSH terminal
  	# Pull and verify line endings immediately
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Check line endings
  	file entrypoint.sh
  	# MUST show: entrypoint.sh: ASCII text
  	# BAD: entrypoint.sh: ASCII text, with CRLF line terminators
  
  	# Verify script has no syntax errors (dry run — does not execute)
  	sh -n entrypoint.sh
  	# No output = no syntax errors
  
  	# Verify it is executable (chmod needed after sed fix)
  	chmod +x entrypoint.sh

STEP 5: Create .dockerignore

  	Tell Docker what NOT to copy into the image
  
  	What this does
  	.dockerignore works exactly like .gitignore but for Docker builds. Without it, Docker copies your 
  	entire venv folder (hundreds of MB), your .env secrets, your SQLite database, and all other junk into the image. 
  	This makes images massive, slow to build, and potentially leaks your secrets into the image layers.
  
  	On Windows — VS Code terminal
  	# Create .dockerignore in your project root (same folder as Dockerfile)
  
  	# Python virtual environment — never copy this
  	# The container installs its own packages from requirements.txt
  	venv/
  
  	# Git history — not needed inside a container
  	.git/
  	.gitignore
  
  	# Python bytecode — regenerated automatically
  	__pycache__/
  	*.pyc
  	*.pyo
  	*.pyd
  
  	# NEVER copy secrets into an image
  	.env
  	.env.*
  
  	# SQLite database — container uses PostgreSQL
  	db.sqlite3
  	*.sqlite3
  
  	# Django generated — regenerated by entrypoint.sh on startup
  	staticfiles/
  	media/
  
  	# Docker config files — no need to copy these into the image
  	Dockerfile
  	docker-compose.yml
  	nginx/
  
  	# OS junk
  	.DS_Store
  	Thumbs.db
  
  	# Editor
  	.vscode/
  
  	# Logs
  	*.log
  
  	# Push .dockerignore
  	git add .dockerignore
  	git commit -m "Add .dockerignore"
  	git push origin main
  
  	On VM — SSH terminal
  	# Pull and verify image size improvement
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Rebuild the test image with .dockerignore in place
  	docker build -t portfolio-test .
  
  	# Check image size
  	docker images portfolio-test
  	# Should be around 300-400MB
  	# Without .dockerignore it would be 1GB+ due to venv being copied
  	# Verify .env did not make it into the image
  
  	docker run --rm --entrypoint="" portfolio-test find /app -name ".env" 2>/dev/null
  	# Should return nothing — .env must not be in the image
  
  	# Clean up
  	docker rmi portfolio-test


STEP 6: Create nginx/nginx.conf

  	Nginx container config for reverse proxy and static files 
  
  	What this does
  	In Phase 1 Nginx talked to Gunicorn through a Unix socket file. In Docker they are separate containers —
  	 Nginx reaches Gunicorn over Docker's internal network using the service name 'web' as the hostname. 
  	 Docker's DNS resolves 'web' to the web container's IP automatically. Static and media files are served 
  	 from shared Docker volumes that both the web and nginx containers mount.
  	 
  	On Windows — VS Code terminal
  	# Create a folder called 'nginx' in your project root
  	# Inside it create 'nginx.conf'
  	# Full path: nginx/nginx.conf
  
  	upstream django {
  		server web:8000;
  		# 'web' = the Docker Compose service name for your Django container
  		# Docker internal DNS resolves 'web' → web container IP
  		# No hardcoded IPs needed — Docker handles it
  	}
  
  	server {
  		listen 80;
  		server_name 192.168.1.105;
  		# Replace with your actual VM IP address
  
  		client_max_body_size 20M;
  		# Your portfolio has image uploads and CV file uploads
  		# 20MB covers most document and image uploads
  		# Increase if you upload larger files
  
  		location = /favicon.ico {
  			access_log off;
  			log_not_found off;
  			# Suppress favicon logs — browsers request it constantly
  			# flooding your logs with useless 404 entries
  		}
  
  		location /static/ {
  			alias /app/staticfiles/;
  			# alias: /static/pages/index.css → /app/staticfiles/pages/index.css
  			# Files come from static_volume shared with web container
  			# collectstatic in entrypoint.sh writes here on every startup
  			expires 1d;
  			add_header Cache-Control "public, immutable";
  			# Tell browsers to cache static files for 1 day
  			# Safe because collectstatic regenerates with fresh content
  		}
  
  		location /media/ {
  			alias /app/media/;
  			# User uploaded files: project images, CV PDFs etc
  			# Files come from media_volume shared with web container
  		}
  
  		location / {
  			proxy_pass http://django;
  			# Forward everything else to Gunicorn in the web container
  
  			proxy_set_header Host $host;
  			# Tell Django the original hostname the browser used
  
  			proxy_set_header X-Real-IP $remote_addr;
  			# Tell Django the real client IP (not Nginx's container IP)
  
  			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  			proxy_set_header X-Forwarded-Proto $scheme;
  			proxy_redirect off;
  
  			proxy_connect_timeout 60s;
  			proxy_read_timeout 60s;
  			# How long Nginx waits for Django to respond
  			# 60s is generous — tune down once everything is stable
  		}
  	}
  
  	# Push nginx config
  	git add nginx/
  	git commit -m "Add Nginx configuration"
  	git push origin main
  
  	On VM — SSH terminal
  	# Pull and validate nginx config syntax
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Spin up a throwaway nginx container just to test config syntax
  	# --rm = delete container after it exits
  	# -v mounts your config file into the container
  	# nginx -t = test config only, do not start nginx
  	docker run --rm   -v $(pwd)/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro   nginx:alpine nginx -t
  
  	# Expected output:
  	# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  	# nginx: configuration file /etc/nginx/nginx.conf test is successful

STEP 7: Update .env on the VM
  
  	Add PostgreSQL credentials  (done on the VM directly)
  
  	What this does
  	The .env file is the ONE file you create and edit directly on the VM — never on Windows, never pushed to GitHub.
  	 It contains your secrets. You need to add PostgreSQL credentials to it now. The DB_PASSWORD and POSTGRES_PASSWORD must be identical —
  	 they are the same password, one read by Django, one read by the Postgres container to create the database on first run.
  	
  	On VM — SSH terminal
  	# Generate a strong random password for the database
  	openssl rand -base64 32
  	# Copy the output — use it for BOTH DB_PASSWORD and POSTGRES_PASSWORD
  	# Edit your .env file
  
  	vim /srv/portfoliosite/portfolio/.env
  
  	# Add or update these lines — keep all your existing variables
  	# and ADD the new PostgreSQL ones:
  
  	SECRET_KEY=your-existing-secret-key
  	DEBUG=False
  	ALLOWED_HOSTS=192.168.1.105,localhost,127.0.0.1
  
  	# PostgreSQL — Django uses these to connect to the database
  	DB_NAME=portfolio_db
  	DB_USER=portfolio_user
  	DB_PASSWORD=paste-your-generated-password-here
  	DB_HOST=db
  	DB_PORT=5432
  
  	# PostgreSQL — Postgres container uses these on FIRST RUN ONLY
  	# to create the database and user
  	# MUST be identical to DB_NAME, DB_USER, DB_PASSWORD above
  	POSTGRES_DB=portfolio_db
  	POSTGRES_USER=portfolio_user
  	POSTGRES_PASSWORD=paste-your-generated-password-here
  
  	# Email (keep your existing values)
  	EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
  	EMAIL_HOST=smtp.gmail.com
  	EMAIL_PORT=587
  	EMAIL=your-gmail@gmail.com
  	EMAIL_PASSWORD=your-gmail-app-password
  
  	# Social links (keep your existing values)
  	LINKEDIN_URL=https://linkedin.com/in/yourprofile
  	FACEBOOK_URL=https://facebook.com/yourprofile
  	INSTAGRAM_URL=https://instagram.com/yourprofile
  	GITHUB_URL=https://github.com/yourusername
  	# Verify the file looks right
  	cat /srv/portfoliosite/portfolio/.env
  	# Confirm DB_PASSWORD and POSTGRES_PASSWORD are the same value
		
STEP 8: Create docker-compose.yml

  	Define all three containers 
  
  	What this does
  	docker-compose.yml is the master file that defines your entire stack.
  	 Three services: db (PostgreSQL), web (Django+Gunicorn), nginx (reverse proxy). 
  	 They communicate over a private Docker network using service names as hostnames. 
  	 Named volumes persist your database and uploaded files independently of containers — you can delete and recreate containers without losing data.
  
  	On Windows — VS Code terminal
  	# Create docker-compose.yml in your project root (same folder as manage.py)
  
  	services:
  
  	  db:
  		image: postgres:16-alpine
  		# postgres:16-alpine = PostgreSQL 16 on Alpine Linux
  		# alpine base = much smaller image, same functionality
  		env_file:
  		  - .env
  		# Reads POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD from .env
  		# Postgres uses these to create the database on FIRST RUN only
  		# On subsequent runs it just opens the existing database from volume
  		volumes:
  		  - postgres_data:/var/lib/postgresql/data
  		  # Named volume — ALL database data lives here
  		  # Survives container deletion and recreation
  		  # docker compose down does NOT delete this
  		  # docker compose down -v DOES delete this — be careful
  		healthcheck:
  		  test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
  		  interval: 5s
  		  timeout: 5s
  		  retries: 5
  		  # pg_isready = built-in postgres command that checks if DB accepts connections
  		  # Docker waits for this to pass before starting the web container
  		  # Without this: web starts, tries to migrate before postgres is ready, crashes
  		restart: unless-stopped
  		# Restart automatically on crash or VM reboot
  		# unless-stopped = only stays stopped if YOU ran docker compose down
  
  	  web:
  		build: .
  		# Build using the Dockerfile in the current directory
  		env_file:
  		  - .env
  		# Reads SECRET_KEY, DEBUG, ALLOWED_HOSTS, DB_* from .env
  		volumes:
  		  - static_volume:/app/staticfiles
  		  # collectstatic in entrypoint.sh writes CSS/JS here
  		  # nginx reads from this same volume to serve static files
  		  - media_volume:/app/media
  		  # User uploaded files: project images, CV PDFs etc
  		  # nginx reads from this same volume to serve media files
  		depends_on:
  		  db:
  			condition: service_healthy
  		  # Only start web AFTER db passes its health check
  		  # This prevents migration errors on first run
  		restart: unless-stopped
  		# NOTE: web does NOT expose any ports to the outside
  		# Only nginx talks to web — through Docker's internal network
  		# This is intentional — Django should never be publicly exposed
  
  	  nginx:
  		image: nginx:alpine
  		ports:
  		  - "80:80"
  		  # Map port 80 on the VM to port 80 in the nginx container
  		  # Format: "HOST_PORT:CONTAINER_PORT"
  		  # This is the ONLY container that exposes a port externally
  		volumes:
  		  - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
  		  # Mount your nginx.conf into the container
  		  # :ro = read-only, nginx can read but not modify it
  		  # ./nginx/nginx.conf = file on VM disk
  		  # /etc/nginx/conf.d/default.conf = where nginx looks for config
  		  - static_volume:/app/staticfiles:ro
  		  # Same volume as web container — nginx reads, web writes
  		  - media_volume:/app/media:ro
  		  # Same volume as web container — nginx reads, web writes
  		depends_on:
  		  - web
  		restart: unless-stopped
  
  	volumes:
  	  postgres_data:
  	  static_volume:
  	  media_volume:
  	  # Named volumes — Docker manages their location on the VM filesystem
  	  # They live at: /var/lib/docker/volumes/
  	  # They persist across: container restarts, docker compose down
  	  # They are deleted by: docker compose down -v
  	  
  	# Push docker-compose.yml
  	git add docker-compose.yml
  	git commit -m "Add docker-compose.yml — complete stack definition"
  	git push origin main
  
  	On VM — SSH terminal
  	# Pull the final file
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  
  	# Validate docker-compose.yml syntax
  	# This also substitutes .env variables and shows the resolved config
  	docker compose config
  	# If this prints the full config without errors — file is valid
  	# If there is a YAML error it tells you the exact line
  
  	=====>> Faced bugs in this step as version was obselete and gunicorn was not mentioned in the requirements.txt

STEP 9: First docker compose up — bring everything online

  	The moment all containers start together for the first time
  
  	What this does
  	This is the first time all three containers run together. Run WITHOUT -d (detached) 
  	first so you see all output live. The first build downloads base images and takes 3-5 minutes. 
  	Watch the output carefully — each line tells you what is happening. After this succeeds you run it detached for production use.
  
  	On VM — SSH terminal
  	# Make sure you are in the project root
  	cd /srv/portfoliosite/portfolio
  
  	# Verify all files are present before building
  	ls
  	# Must see: Dockerfile, .dockerignore, docker-compose.yml,
  	#           entrypoint.sh, .env, manage.py, nginx/, requirements.txt
  	# Build and start — without -d so you see all output live
  	docker compose up --build
  
  	# You will see Docker building your image layer by layer:
  	# [+] Building...
  	#  => FROM python:3.12-slim          ← downloading base image
  	#  => apt-get install libpq-dev gcc  ← installing system deps
  	#  => pip install -r requirements    ← installing Python packages
  	#  => COPY . .                       ← copying your code
  	#
  	# Then containers starting:
  	# ✔ Container portfolio-db-1     Started
  	# ✔ Container portfolio-web-1    Started
  	# ✔ Container portfolio-nginx-1  Started
  	#
  	# Then live logs:
  	# db-1   | database system is ready to accept connections
  	# web-1  | === Running database migrations ===
  	# web-1  | Operations to perform: Apply all migrations
  	# web-1  | Running migrations: OK
  	# web-1  | === Collecting static files ===
  	# web-1  | 140 static files copied to '/app/staticfiles'
  	# web-1  | === Starting Gunicorn ===
  	# web-1  | [INFO] Listening at: http://0.0.0.0:8000
  	# Once you see Gunicorn started:
  	# Open browser on Windows: http://192.168.1.105
  	# Your portfolio should load with full CSS and styling
  
  	# Press Ctrl+C to stop all containers
  	# Now run in detached mode (background)
  	docker compose up -d
  
  	# Check all containers are running
  	docker compose ps
  	# All three should show Status: running
  
  	===> If somtthing failed run correct the mistekes and run commands:
  		docker compose down (delets all the containers)
  		docker build --no-cache (Rebuild with no cache — forces pip install to run fresh)
  		docker compose up / docker compose up -d 
  		
STEP 10: Create superuser and test everything

  	Final verification — superuser, admin, uploads, reboot test
  
  	What this does
  	Your PostgreSQL database is fresh and empty. Create the superuser to access Django admin.
  	 Then test the full stack including media file uploads (since your project has image and CV uploads).
  	 Finally reboot the VM to confirm everything auto-starts correctly.
  	 
  	On VM — SSH terminal
  	# Create Django superuser inside the running web container
  	docker compose exec web python manage.py createsuperuser
  	# docker compose exec web = run this command inside the web container
  	# Fill in username, email, password when prompted
  	# Test the admin works
  	# Visit: http://192.168.1.105/admin
  	# Login with your new superuser credentials
  	# Add a test portfolio project with an image to verify media uploads work
  	
  	# Check media uploads are persisting in the volume
  	docker volume inspect portfolio_media_volume
  	# Shows the volume location on disk
  
  	# List files uploaded via admin
  	docker compose exec web ls /app/media/
  	# Should show your uploaded files
  
  	# Test auto-restart — reboot the VM
  	sudo reboot
  
  	# Wait 60 seconds then SSH back in
  	ssh kaushal20@192.168.1.105
  
  	cd /srv/portfoliosite/portfolio
  	docker compose ps
  	# All three containers should be running automatically
  	# without you running any commands
  
  	# Verify site is still up
  	# http://192.168.1.105 should load immediately

FINAL TIP: Daily workflow and essential commands

  	Everything you need for day-to-day development from here
  
  	What this does
  	This is your reference for the commands you will use constantly from now on. 
  	The deploy workflow is the three-command sequence you will run on the VM every time you push new code — until you automate it with CICD.
  
  	On Windows — VS Code terminal
  	# After making any code change on Windows:
  	git add .
  	git commit -m "Your descriptive message"
  	git push origin main
  
  	On VM — SSH terminal
  	# Manual deploy after every push :
  	cd /srv/portfoliosite/portfolio
  	git pull origin main
  	docker compose up -d --no-deps --build web
  	# --no-deps = don't restart db and nginx (only web changed)
  	# --build = rebuild the web image with your new code
  	# -d = run in background
  	# After adding a new Django model or changing an existing one:
  
  	docker compose exec web python manage.py migrate
  	# Watch live logs (very useful for debugging):
  	docker compose logs -f web      # Django/Gunicorn logs
  	docker compose logs -f nginx    # Nginx access logs
  	docker compose logs -f db       # Postgres logs
  	docker compose logs -f          # All containers together
  	# Get a shell inside the web container:
  
  	docker compose exec web sh
  	# From here you can run manage.py commands, inspect files,
  	# check environment variables with: env | grep DB
  	# Exit with: exit
  
  	# Check container status
  	docker compose ps
  
  	# Check resource usage (CPU and RAM per container)
  	docker stats
  
  	# Stop everything (keeps all data in volumes)
  	docker compose down

	# DANGER: Stop AND delete all data including database
	# Only use this if you want to completely start fresh
	docker compose down -v
