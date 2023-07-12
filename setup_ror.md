# Rails Application Deployment Guide

This guide provides step-by-step instructions for deploying a Ruby on Rails application using AWS EC2, Nginx, Passenger, Redis, and MySQL RDS. Follow the instructions below to set up and deploy your Rails application.

## Prerequisites

- An AWS account with access to EC2, RDS, and other necessary services.
- A domain name or IP address for your application.

## Step 1: Set up AWS EC2 Instance

1. Launch an EC2 instance with Ubuntu or your preferred operating system.
2. Configure the necessary security groups to allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS).
3. Note down the public IP address or DNS name of the EC2 instance.

## Step 2: SSH into the EC2 Instance

1. Open your terminal and run the following command, replacing `your-key.pem` with the path to your AWS key file and `<EC2-Instance-Public-IP>` with the public IP address of your EC2 instance:

```shell
ssh -i your-key.pem ubuntu@<EC2-Instance-Public-IP>
```

## Step 3: Install necessary dependencies

1. Update the system's package lists and install necessary dependencies:

```shell
sudo apt update
sudo apt install -y curl gnupg2 dirmngr
```

## Step 4: Install Ruby and Rails dependencies

1. Install required libraries and dependencies:

```shell
sudo apt install -y libssl-dev libreadline-dev zlib1g-dev
sudo apt install -y libsqlite3-dev sqlite3
sudo apt install -y libpq-dev # For PostgreSQL
sudo apt install -y libmysqlclient-dev # For MySQL
sudo apt install -y libmagickwand-dev # For ImageMagick
sudo apt install -y build-essential # For compiling Ruby
```

## Step 5: Install RVM and Ruby

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

## Step 6: Install and configure Passenger

1. Install Passenger and Nginx module:

```shell
sudo apt-get install -y libnginx-mod-http-passenger
sudo systemctl restart nginx
```

## Step 7: Set up MySQL RDS

1. Go to the AWS Management Console and navigate to the RDS service.
2. Create a new MySQL database instance and configure it according to your needs.
3. Note down the endpoint, username, password, and database name.

## Step 8: Set up Redis

1. Install Redis:

```shell
sudo apt-get install -y redis-server
```

## Step 9: Clone your Rails application code

1. Clone your Rails application code to the EC2 instance.

## Step 10: Configure your Rails application

1. Update the `config/database.yml` file with the MySQL RDS details:

```yaml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch('RAILS_MAX_THREADS') { 5 } %>
  host: your-rds-endpoint
  username: your-rds-username
  password: your-rds-password
  database: your-rds-database

development:
  <<: *default

test:
  <<: *default

staging:
  <<: *default

production:
  <<: *default
```

2. Update the `config/initializers/redis.rb` file with the Redis configuration:

```ruby
if Rails.env.production?
  uri = URI.parse(ENV['REDIS_URL'])
  Redis.current = Redis.new(host: uri.host, port: uri.port, password: uri.password)
else
  Redis.current = Redis.new(host: 'localhost', port: 6379)
end
```

## Step 11: Install application dependencies

1. Change to your Rails application directory:

```shell
cd your-rails-app
```

2. Install the application dependencies:

```shell
bundle install --deployment --without development test
```

## Step 12: Precompile assets

1. Precompile assets:

```shell
RAILS_ENV=production bundle exec rails assets:precompile
```

## Step 13: Set up Nginx configuration for Passenger

1. Edit the Nginx configuration file:

```shell
sudo nano /etc/nginx/nginx.conf
```

2. Replace the contents of the file with the following:

```nginx
# Add the provided Nginx configuration
```

## Step 14: Update Nginx configuration for your Rails application

1. Create a new Nginx site configuration:

```shell
sudo nano /etc/nginx/sites-available/myapp
```

2. Replace the contents of the file with the following:

```nginx
# Add the provided Nginx site configuration
```

## Step 15: Enable the Nginx site configuration

```shell
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

## Step 16: Restart Nginx

```shell
sudo systemctl restart nginx
```

## Step 17: Start the Rails application

```shell
bundle exec rails db:migrate RAILS_ENV=staging # or production
bundle exec rails assets:precompile RAILS_ENV=staging # or production
bundle exec rails assets:clobber
bundle exec rails tmp:create
RAILS_ENV=staging bundle exec passenger start --port 3000
```

Replace `staging` with `production` if deploying to a production environment.

That's it! Your Ruby on Rails application should now be successfully deployed using Nginx, Passenger, Redis, and MySQL RDS.

Please note that this guide provides a general outline for deployment, and you may need to make additional configurations or adjustments based on your specific application and deployment setup.
