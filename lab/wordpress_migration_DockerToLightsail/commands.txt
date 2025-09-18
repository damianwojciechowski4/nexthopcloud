# Check docker compose stack
## 1.Check existing docker-compose stack
docker-compose ls | grep wordpress

## 2.Check running docker compose processes
docker-compose ps

## 3.Check running docker containers
docker ps -a | egrep 'wordpress|phpmyadmin|mariadb'

#Backup Docker containers data
##  Create a backup folder and navigate into it
mkdir -p ~/wp-backup && cd ~/wp-backup
## Dump the DB from the mariadb container
docker exec -i mariadb \
  mysqldump -u root -psecret --databases wordpress \
  --single-transaction --quick --routines --events --triggers \
  > wordpress.sql
  
  
#Export Wordpress Files and Config
## Method 1
### Archive wp-content from the named volume to the current host dir (~/wp-backup)
docker run --rm --network none --read-only \
  -v wordpress_stack-compose_wordpress_data:/var/www/html:ro \
  -v "$PWD":/backup \
  alpine sh -c 'cd /var/www/html && tar czf /backup/wp-content.tgz wp-content'
  

### Copy wp-config.php into the current host dir (~/wp-backup)
docker run --rm --network none --read-only \
  -v wordpress_stack-compose_wordpress_data:/var/www/html:ro \
  -v "$PWD":/backup \
  alpine sh -c 'cp /var/www/html/wp-config.php /backup/wp-config.php'
  
  
### Copy .htaccess into the current host dir (~/wp-backup)
docker run --rm --network none --read-only \
  -v wordpress_stack-compose_wordpress_data:/var/www/html:ro \
  -v "$PWD":/backup \
  alpine sh -c 'cp /var/www/html/.htaccess /backup/.htaccess'
  
## Method 2
## docker cp syntax
docker cp <Wordpress CONTAINER_NAME or ID>:SOURCE_PATH DEST_PATH

# Organize Backup Files into wp-backup catalog
touch migrationNotes.txt
echo Current WordPress URL:http://$(hostname -I | awk '{print $1}'):$(docker port c2befafabfd1 | grep "0.0.0.0" | awk -F: '{print $2}') > migrationNotes.txt

cat migrationNotes.txt

## Verify that all files are within backup folder
cd ~/wp-backup
tree -a -L 1

├── .htaccess
├── migrationNotes.txt
├── wordpress.sql
├── wp-config.php
└── wp-content

2 directories, 4 files

# Archive catalog 
## -z: compress archive with gzip;
## -c: create a new archive;
## -v: verbose mode to show progress
## -f: speciy archive file name
tar -zcvf wp-backup.tar.gz wp-backup

## Check file spaces for comparison
du -sh wp-backup
### Example: 317M    wp-backup

du -sh wp-backup.tar.gz
### Exammple: 89M     wp-backup.tar.gz

# Configure Lightsail
## Use api request to get your current public IP address
curl https://ipinfo.io/ip

### -> save Public IP result

#Modify key privileges
chmod 400 ~/.ssh/LightsailDefaultKey-eu-central-1.pem

# Login to Lightsail instance 
## where PublicIP = PublicIP of your Lightsail instance
ssh -i ~/.ssh/LightsailDefaultKey-eu-central-1.pem bitnami@<PublicIP>


# Copy folder from Docker-host to Lightsail instance using scp protocol
## where PublicIP = PublicIP of your Lightsail instance
scp -i ~/.ssh/LightsailDefaultKey-eu-central-1.pem \
~/wp-backup.tar.gz bitnami@<PublicIP>:/opt/bitnami/wordpress/wp-backup.tar.gz
#  Output
## wp-backup.tar.gz                                    100%   89MB  12.0MB/s   00:07


# Extract catalog
## -x: extract
## -v: verbose mode to show progress
## -f: specify archite file name
sudo tar -xvf wp-backup.tar.gz
cd wp-backup
ls -l
#  Example
## total 6968
## -rw-rw-r-- 1 bitnami bitnami      47 Sep 14 11:18 migrationNotes.txt
## -rw-rw-r-- 1 bitnami bitnami 7115469 Sep  9 16:45 wordpress.sql
## -rw-r--r-- 1 bitnami bitnami    5919 Jul 28 17:16 wp-config.php
##drwxr-xr-x 7 bitnami bitnami    4096 Sep 15 17:07 wp-content

#Prepare paths to important migration files
sql="/opt/bitnami/wordpress/wp-backup/wordpress.sql"
echo $sql
wpcontent="/opt/bitnami/wordpress/wp-backup/wp-content"
echo $wpcontent
wpconfig="/opt/bitnami/wordpress/wp-backup/wp-config.php"
echo $wpconfig

## show the Bitnami app credentials (contains the DB/root password)
sudo cat /home/bitnami/bitnami_credentials
### password value should be displayed

#Connect to your DB
/opt/bitnami/mariadb/bin/mariadb -u root -p
##Verify configured databases
SHOW DATABASES;
## Remove existing bitnami_wordpress database
DROP DATABASE bitnami_wordpress;
## Create bitnami_wordpress database from the scratch
CREATE DATABASE bitnami_wordpress;
## Exit mysql session
EXIT;

# Make a backup of your original dump first
cp /opt/bitnami/wordpress/wp-backup/wordpress.sql /opt/bitnami/wordpress/wp-backup/wordpress_original.sql

# Edit the dump file
sudo nano /opt/bitnami/wordpress/wp-backup/wordpress.sql

## Verify Changes
head -25 wordpress.sql | grep -E "(Database|CREATE DATABASE|USE)"


# Import modified dump file
/opt/bitnami/mariadb/bin/mariadb -u root -p < $sql

# Verify the import
/opt/bitnami/mariadb/bin/mariadb -u root -p

## Verify SQL DB state
SHOW DATABASES;
USE bitnami_wordpress;
SHOW TABLES;
SELECT  option_name, option_value FROM wp_options WHERE option_name IN ('siteurl', 'home');
EXIT;

# Backup current wp-content
sudo cp -r /opt/bitnami/wordpress/wp-content /opt/bitnami/wordpress/wp-content-original

# Remove default content
sudo rm -rf /opt/bitnami/wordpress/wp-content/uploads
sudo rm -rf /opt/bitnami/wordpress/wp-content/plugins/*
sudo rm -rf /opt/bitnami/wordpress/wp-content/themes/*

# Copy your content including hidden files
sudo cp -r $wpcontent/. /opt/bitnami/wordpress/wp-content/

# Fix permissions
sudo chown -R bitnami:daemon /opt/bitnami/wordpress/wp-content
sudo chmod -R 755 /opt/bitnami/wordpress/wp-content
sudo find /opt/bitnami/wordpress/wp-content -type f -exec chmod 644 {} \;


# Update WordPress URLs in wp-config.php
nano -l /opt/bitnami/wordpress/wp-config.php


# Update URLs in the DATABASE
cd /opt/bitnami/wordpress
# sudo wp search-replace 'old-string' 'new-string' --allow-root
## First Dry Run
sudo wp search-replace 'http://10.130.0.17:8082' 'https://nexthopcloud.com' --all-tables --allow-root --dry-run

# Replacement 
sudo wp search-replace 'http://10.130.0.17:8082' 'https://nexthopcloud.com' --all-tables --allow-root

## Check if any old URLs remained
sudo wp db search '10.130.0.17' --allow-root

# Verify if the values are updated
sudo wp option get siteurl --allow-root
sudo wp option get home --allow-root


# Login to DB
/opt/bitnami/mariadb/bin/mariadb -u root -p bitnami_wordpress
#SQL
-- Find all references to the old URL
SELECT * FROM wp_options WHERE option_value LIKE '%10.130.0.17:8082%';
SELECT * FROM wp_postmeta WHERE meta_value LIKE '%10.130.0.17:8082%';
SELECT * FROM wp_posts WHERE post_content LIKE '%10.130.0.17:8082%';

-- Update any found references (be careful with serialized data)
UPDATE wp_options SET option_value = REPLACE(option_value, 'http://10.130.0.17:8082', 'https://nexthopcloud.com') WHERE option_value LIKE '%10.130.0.17:8082%';
UPDATE wp_postmeta SET meta_value = REPLACE(meta_value, 'http://10.130.0.17:8082', 'https://nexthopcloud.com') WHERE meta_value LIKE '%10.130.0.17:8082%';
UPDATE wp_posts SET post_content = REPLACE(post_content, 'http://10.130.0.17:8082', 'https://nexthopcloud.com') WHERE post_content LIKE '%10.130.0.17:8082%';

# Force Elementor to Regenerate everything

## Clear all Elementor data and force regeneration
wp option delete elementor_css_print_method --allow-root
wp option delete elementor_frontend_config --allow-root
wp option update elementor_clear_cache 1 --allow-root

## Force WordPress to regenerate permalinks
wp rewrite flush --allow-root

## Clear all caches
wp cache flush --allow-root

# Restart Apache to ensure all changes take effect
sudo /opt/bitnami/ctlscript.sh restart apache
