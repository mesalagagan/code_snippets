
# Ruby on Rails Application Deployment Guide

This guide provides step-by-step instructions for deploying a Ruby on Rails application using AWS EC2, Nginx, Passenger, Redis, and MySQL RDS.

## Prerequisites

- An AWS account with access to EC2, RDS, and other necessary services.
- A domain name or IP address for your application.

## Deployment Steps

### Step 1: Set up AWS EC2 Instance

1. Launch an EC2 instance with Ubuntu or your preferred operating system.
2. Configure the necessary security groups to allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS).
3. Note down the public IP address or DNS name of the EC2 instance.

### Step 2: SSH into the EC2 Instance

1. Open your terminal and run the following command, replacing `your-key.pem` with the path to your AWS key file and `<EC2-Instance-Public-IP>` with the public IP address of your EC2 instance:

   ```shell
   ssh -i your-key.pem ubuntu@<EC2-Instance-Public-IP>
   ```

### Step 3: Install necessary dependencies

1. Update the system's package lists and install necessary dependencies:

   ```shell
   sudo apt update
   sudo apt install -y curl gnupg2 dirmngr
   ```

### Step 4: Install Ruby and Rails dependencies

1. Install required libraries and dependencies:

   ```shell
   sudo apt install -y libssl-dev libreadline-dev zlib1g-dev
   sudo apt install -y libsqlite3-dev sqlite3
   sudo apt install -y libpq-dev # For PostgreSQL
   sudo apt install -y libmysqlclient-dev # For MySQL
   sudo apt install -y libmagickwand-dev # For ImageMagick
   sudo apt install -y build-essential # For compiling Ruby
   ```

### Step 5: Install RVM and Ruby

1. Install RVM (Ruby Version Manager) and Ruby:

   ```shell
   sudo apt install -y gpg
   curl -sSL https://rvm.io/mpapis.asc | sudo gpg --import -
   curl -sSL https://get.rvm.io | sudo bash -s stable
   source /etc/profile.d/rvm.sh
   rvm install 2.7.4
   rvm use 2.7.4 --default
   gem install bundler
   ```

### Step 6: Install and configure Passenger

1. Install Passenger and Nginx module:

   ```shell
   sudo apt-get install -y libnginx-mod-http-passenger
   sudo systemctl restart nginx
   ```

### Step 7: Set up MySQL RDS

1. Go to the AWS Management Console and navigate to the RDS service.
2. Create a new MySQL database instance and configure it according to your needs.
3. Note down the endpoint, username, password, and database name.

### Step 8: Set up Redis

1. Install Redis:

   ```shell
   sudo apt-get install -y redis-server
   ```

### Step 9: Clone your Rails application code

1. Clone your Rails application code to the EC2 instance.

### Step 10: Configure your Rails application

1. Update the `config/database.yml` file with the MySQL RDS details.

2. Update the `config/initializers/redis.rb` file with the Redis configuration.

### Step 11: Install application dependencies

1. Change to your Rails application directory:

   ```shell
   cd your-rails-app
   ```

2. Add below gems into Gemfile:
   ```shell
   source 'https://rubygems.org'
  
   gem 'rails', '6.1.4'
   gem 'redis', '~> 4.2'
   gem 'sidekiq', '~> 6.2'
   gem 'passenger', '~> 6.0', '>= 6.0.11'
  
   ```
   
3. Install the application dependencies:

   ```shell
   bundle install --deployment --without development test
   ```

### Step 12: Precompile assets

1. Precompile assets:

   ```shell
   RAILS_ENV=production bundle exec rails assets:precompile
   ```

### Step 13: Set up Nginx configuration for Passenger

1. Edit the Nginx configuration file:

   ```shell
   sudo nano /etc/nginx/nginx.conf
   
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    include /etc/nginx/modules-enabled/*.conf;
    
    events {
        worker_connections 768;
        # Multi_accept on;
    }
    
    http {
        ##
        # Basic Settings
        ##
    
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
    
        server_tokens off;
    
        # MIME
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
    
        ##
        # SSL Settings
        ##
    
        ssl_protocols TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;
    
        ##
        # Logging Settings
        ##
    
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
    
        ##
        # Gzip Settings
        ##
    
        gzip on;
        gzip_disable "msie6";
    
        ##
        # Virtual Host Configs
        ##
    
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;

        ##
        # Passenger Configuration
        ##

        passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
        passenger_ruby /usr/local/rvm/gems/ruby-2.7.4/wrappers/ruby;
    }
   ```

2. Replace the contents of the file with the provided Nginx configuration.

### Step 14: Update Nginx configuration for your Rails application

1. Create a new Nginx site configuration:

   ```shell
   sudo nano /etc/nginx/sites-available/myapp
   ```

   ```shell
   
    server {
        listen 80;
        server_name localhost;
        #server_name your-domain.com; # Replace with your domain or IP address
    
        root /home/ubuntu/your-rails-app/public; # Replace with the path to your Rails app
    
        passenger_enabled on;
        passenger_app_env staging; # Adjust the environment (staging/production)
    
        location /cable {
            passenger_app_group_name your-rails-app_websocket;
            passenger_force_max_concurrent_requests_per_process 0;
        }

        location / {
            proxy_pass http://localhost:3000; # Adjust the port if necessary
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }        

        ##
        # Sidekiq Configuration
        ##
    
        upstream sidekiq {
            server 127.0.0.1:5000; # Adjust the port if necessary
        }
   
        location /sidekiq {
            alias /home/ubuntu/your-rails-app/public; # Replace with the path to your Rails app
            passenger_app_group_name your-rails-app_sidekiq;
            passenger_force_max_concurrent_requests_per_process 1;
        }
    }

   ```

### Step 15: Enable the Nginx site configuration

   ```shell
   sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
   ```

### Step 16: Restart Nginx

   ```shell
   sudo systemctl restart nginx
   ```

### Step 17: Start the Rails application

   ```shell
   bundle exec rails db:migrate RAILS_ENV=staging # or production
   bundle exec rails assets:precompile RAILS_ENV=staging # or production
   bundle exec rails assets:clobber
   bundle exec rails tmp:create
   RAILS_ENV=staging bundle exec passenger start --port 3000
   ```

Replace `staging` with `

production` if deploying to a production environment.

That's it! Your Ruby on Rails application should now be successfully deployed using Nginx, Passenger, Redis, and MySQL RDS.
