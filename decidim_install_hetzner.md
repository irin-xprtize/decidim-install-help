# Decidim installation 
## Notes
### For Hetzner cloud

* Install Ubuntu 20.04 on PC
* Register an account and set up server in Hetzner cloud. Reference: https://docs.hetzner.com/cloud/servers/getting-started/creating-a-server
Choose Ubuntu as the OS
Dedicated server
Server???
Volume???
Network???
Firewall???
Additional features - userdata, backups, placement groups???
Set up SSH keys on the PC and copy/paste to the server registration panel
```
sshkeygen
cat ~/.ssh/id_rsa.pub
```
* Note the server IP (assume it as 1.2.3.4) and connect with the server from command line and login as the root user
* Point A record of the your domain (decidim-xprtize.org) to the server IP
* Give a name to the server
```
ssh root@1.2.3.4
```
* Create a non-root user with administrative privileges. Reference: https://gorails.com/deploy/ubuntu/20.04
  https://github.com/Platoniq/decidim-install/blob/main/decidim-focal.md

  https://www.youtube.com/watch?v=-sqXXNEzeCw
```
adduser deploy 
adduser deploy sudo
exit
```
* Copy SSH keys for the non-root user to work
```
ssh-copy-id root@1.2.3.4
ssh-copy-id deploy@1.2.3.4
```
* Log in as "deploy" user
```
ssh deploy@1.2.3.4
```
* Stay logged in as this user for the rest of the installation
* Enable Firewall
```
sudo ufw status

sudo ufw default allow outgoing
sudo ufw default deny incoming

sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
```
* Keep system up-to-date
```
sudo apt update
sudo apt upgrade
sudo apt autoremove
```
* Configure timezone of the server
```
sudo dpkg-reconfigure tzdata
```
* Install required packages
```
sudo apt install autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev
```
* Install Ruby using rbenv method
```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
```
* Check rbenv is correctly installed
```
type rbenv
```
* Install ruby and verify
```
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
rbenv install 2.6.3
rbenv global 2.6.3
ruby -v
```
* Set up Gem
```
echo "gem: --no-document" > ~/.gemrc
gem install bundler
```
* Test everything is fine
```
gem env home
```
* Install Postgresql for database and install other dependencies
```
sudo apt install -y postgresql libpq-dev
sudo apt install -y nodejs imagemagick libicu-dev
```
* Install decidim
```
gem install decidim
decidim decidim-xprtize
```
* Create postgreql database on the server
Install
```
sudo apt-get install postgresql postgresql-contrib libpq-dev
```
Login to database server
```
sudo su - postgres
```
Create user "deploy" and "decidim-xprtize" database
```
createuser --pwprompt deploy
createdb -O deploy decidim-xprtize
```
And exit
```
exit
```
Configure database.yml file???
* Change to the app folder
```
cd decidim-xprtize
```
* Use figaro gem to take care of secrets; also add other gems (required later)
* Change Gemfile
```
nano Gemfile
```
* Add following to the Gemfile
```
gem "figaro"

group :production do
  gem "passenger"
  gem 'delayed_job_active_record'
  gem "daemons"
end
```
* Install the gems
```
bundle install
```
* Configure git, create a new repository, add SSH/token and push everything to Github
```
git init
git add .
git commit -m "First commit"
git remote add origin ---
git push -u origin
```
* Create first admin user
```
bin/rails console -e production
```
Add the following in the prompt that appears
```
email = "my-admin@email"
password = "<a secure password>"
user = Decidim::System::Admin.new(email: email, password: password, password_confirmation: password)
user.save!
```
Write "quit" or press ctrl+D to exit rails console.

* Install nginx for webserver
```
sudo apt -y install nginx
sudo ufw allow http
sudo ufw allow https
```
* Install the gateway, passenger
```
sudo apt install -y dirmngr gnupg
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo apt install -y apt-transport-https ca-certificates
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger focal main > /etc/apt/sources.list.d/passenger.list'
sudo apt update
sudo apt install -y libnginx-mod-http-passenger

```
* Activate passenger (on command line )
```
if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi
```
* Check if installation is proper (it has to list the file)
```
sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
```
* Restart nginx
```
sudo service nginx restart
```
* Make sure passenger points to correct Ruby version. Open the mod-http-passenger.conf file and edit it (if needed)
```
sudo nano /etc/nginx/conf.d/mod-http-passenger.conf
```
* The file must look like below finally.
```
### Begin automatically installed Phusion Passenger config snippet ###
passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /home/decidim/.rbenv/shims/ruby;
### End automatically installed Phusion Passenger config snippet ###
```
* Validate the installation
```
passenger-config validate-install
```
* Create new nginx config file for our application
```
sudo nano /etc/nginx/sites-enabled/decidim.conf
```
* Paste the following inside the file 
```
server {
    listen 80;
    listen [::]:80 ipv6only=on;

    server_name decidim-xprtize.org;
    client_max_body_size 32M;

    passenger_enabled on;
    passenger_ruby /home/decidim/.rbenv/shims/ruby;

    rails_env    production;
    root         /home/decidim/decidim-xprtize/public;
}
```
* Update Github
```
git add .
git commit -m "First commit"
git push -u origin
```
*  Restart nginx
```
sudo service nginx restart
```

* Set up geolocation service 
Create an account in Here Maps
Register a developer account
https://developer.here.com/?create=Evaluation&keepState=true&step=account
```
cd decidim-xprtize
```
Edit application.yml file to add App code
```
nano ~/decidim-app/config/application.yml
```
Add following line in the file
```
GEOCODER_LOOKUP_API_KEY: <your-App-Code>
```
Include this in .gitignore to keep the credentials secret
```
echo "/config/application.yml" >> ~/decidim-xprtize/.gitignore
```
Edit secrets.yml file
```
nano ~/decidim-app/config/secrets.yml
```

Add following
```
...
  geocoder:
    here_api_key: <%= ENV["GEOCODER_LOOKUP_API_KEY"] %>
...
```
Include this in .gitignore to keep the credentials secret
```
echo "/config/secrets.yml" >> ~/decidim-xprtize/.gitignore
```
Uncomment following lines in the opened file
```
nano config/initializers/decidim.rb
```

```
  config.geocoder = {
    static_map_url: "https://image.maps.ls.hereapi.com/mia/1.6/mapview",
    here_api_key: Rails.application.secrets.geocoder[:here_api_key]
  }
```
 Update Github
```
git add .
git commit -m "First commit"
git push -u origin
```
Restart passenger
```
sudo passenger-config restart-app ~/decidim-xprtize
```
* Enable SSL
Use Let's Encrypt for test
```
sudo add-apt-repository -y ppa:certbot/certbot
sudo apt install -y python-certbot-nginx
```

```
sudo certbot --nginx -d decidim-xprtize.org
```
* Set up email (Gmail for test)

 Can be configured on the admin panel

 Create application specific password in the Gmail account

 ![smtp](C:\Users\irins\Downloads\smtp.png)

* Update Github
```
git add .
git commit -m "First commit"
git push -u origin
```
* Ready to use???