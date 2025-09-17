# Apache Guacamole Migration Guide: CentOS 7 to Ubuntu 24.04 LTS

## Migration Overview
This guide covers migrating Apache Guacamole from:
- **Source**: CentOS 7 with Guacamole 1.1.0
- **Target**: Ubuntu 24.04 LTS with Guacamole 1.6.0

## Prerequisites
- Root access to both servers
- Network connectivity between servers
- Sufficient disk space on the new server
- Database access credentials from the old server

---

## Phase 1: Prepare the Old Server (CentOS 7)

### Step 1.1: Document Current Configuration
```bash
# Create documentation directory
sudo mkdir -p /tmp/guacamole-migration
cd /tmp/guacamole-migration

# Document system information
echo "=== Current Guacamole Configuration ===" > migration-info.txt
echo "Date: $(date)" >> migration-info.txt
echo "Server: $(hostname)" >> migration-info.txt
echo "OS: $(cat /etc/redhat-release)" >> migration-info.txt
echo "" >> migration-info.txt

# Document Guacamole version
echo "=== Guacamole Version ===" >> migration-info.txt
# Check web interface or JAR files for version
find /var/lib/tomcat* /usr/share/tomcat* -name "guacamole-*.jar" 2>/dev/null >> migration-info.txt
echo "" >> migration-info.txt

# Document database configuration
echo "=== Database Configuration ===" >> migration-info.txt
if [ -f /etc/guacamole/guacamole.properties ]; then
    grep -E "(mysql|postgresql)" /etc/guacamole/guacamole.properties >> migration-info.txt
fi
echo "" >> migration-info.txt

# Document services
echo "=== Running Services ===" >> migration-info.txt
sudo systemctl status guacd >> migration-info.txt 2>&1
sudo systemctl status tomcat >> migration-info.txt 2>&1
echo "" >> migration-info.txt

# Document network configuration
echo "=== Network Configuration ===" >> migration-info.txt
sudo netstat -tlnp | grep -E "(8080|4822)" >> migration-info.txt
```

### Step 1.2: Export Database
#### For MySQL:
```bash
# Export database structure and data
mysqldump -u root -p --single-transaction --routines --triggers guacamole > guacamole-export.sql

# Verify export
ls -la guacamole-export.sql
echo "Database export lines: $(wc -l < guacamole-export.sql)"
```

#### For PostgreSQL:
```bash
# Export database
sudo -u postgres pg_dump guacamole > guacamole-export.sql

# Verify export
ls -la guacamole-export.sql
echo "Database export lines: $(wc -l < guacamole-export.sql)"
```

### Step 1.3: Backup Configuration Files
```bash
# Copy all Guacamole configuration
sudo cp -r /etc/guacamole/ ./guacamole-config/

# Backup extensions and libraries
mkdir -p extensions lib
if [ -d /etc/guacamole/extensions ]; then
    sudo cp /etc/guacamole/extensions/* ./extensions/ 2>/dev/null || true
fi
if [ -d /etc/guacamole/lib ]; then
    sudo cp /etc/guacamole/lib/* ./lib/ 2>/dev/null || true
fi

# Document what we found
echo "=== Configuration Files Backed Up ===" >> migration-info.txt
ls -la guacamole-config/ >> migration-info.txt
echo "" >> migration-info.txt
echo "=== Extensions Found ===" >> migration-info.txt
ls -la extensions/ >> migration-info.txt
echo "" >> migration-info.txt
echo "=== Libraries Found ===" >> migration-info.txt
ls -la lib/ >> migration-info.txt
```

### Step 1.4: Export User Data and Connections
```bash
# Create SQL scripts to export user data
cat > export-users.sql << 'EOF'
-- Export users and their data for migration
SELECT 'Users found:' as info, COUNT(*) as count FROM guacamole_user;
SELECT 'Connections found:' as info, COUNT(*) as count FROM guacamole_connection;
SELECT 'User groups found:' as info, COUNT(*) as count FROM guacamole_user_group;
EOF

# Run the export
if command -v mysql &> /dev/null; then
    mysql -u root -p guacamole < export-users.sql > user-data-summary.txt
elif command -v psql &> /dev/null; then
    sudo -u postgres psql guacamole < export-users.sql > user-data-summary.txt
fi

cat user-data-summary.txt >> migration-info.txt
```

### Step 1.5: Create Migration Package
```bash
# Create transfer package
tar -czf guacamole-migration-$(date +%Y%m%d).tar.gz \
    migration-info.txt \
    guacamole-export.sql \
    guacamole-config/ \
    extensions/ \
    lib/ \
    user-data-summary.txt

# Verify package
ls -la guacamole-migration-*.tar.gz
echo "Migration package created successfully"
```

---

## Phase 2: Prepare the New Server (Ubuntu 24.04)

### Step 2.1: Update System and Install Dependencies
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y build-essential libcairo2-dev libjpeg-turbo8-dev \
    libpng-dev libtool-bin libossp-uuid-dev libavcodec-dev libavformat-dev \
    libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev \
    libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev \
    libpulse-dev libssl-dev libvorbis-dev libwebp-dev wget tar

# Install Java 11 (required for Guacamole)
sudo apt install -y openjdk-11-jdk

# Verify Java installation
java -version
```

### Step 2.2: Install and Configure Database
#### For MySQL:
```bash
# Install MySQL
sudo apt install -y mysql-server

# Secure MySQL installation
sudo mysql_secure_installation

# Create Guacamole database and user
sudo mysql -u root -p << 'EOF'
CREATE DATABASE guacamole DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER 'guacamole'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON guacamole.* TO 'guacamole'@'localhost';
FLUSH PRIVILEGES;
EXIT;
EOF
```

#### For PostgreSQL:
```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Create Guacamole database and user
sudo -u postgres psql << 'EOF'
CREATE DATABASE guacamole;
CREATE USER guacamole WITH PASSWORD 'StrongPassword123!';
GRANT ALL PRIVILEGES ON DATABASE guacamole TO guacamole;
\q
EOF
```

### Step 2.3: Install Apache Tomcat
```bash
# Create tomcat user
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat

# Download and install Tomcat 10
cd /tmp
wget https://downloads.apache.org/tomcat/tomcat-10/v10.1.28/bin/apache-tomcat-10.1.28.tar.gz
sudo tar -xzf apache-tomcat-10.1.28.tar.gz -C /opt/tomcat --strip-components=1

# Set permissions
sudo chown -R tomcat: /opt/tomcat
sudo chmod +x /opt/tomcat/bin/*.sh

# Create systemd service
sudo tee /etc/systemd/system/tomcat.service << 'EOF'
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Enable and start Tomcat
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat
sudo systemctl status tomcat
```

---

## Phase 3: Install Guacamole 1.6.0

### Step 3.1: Download Guacamole 1.6.0
```bash
cd /tmp
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-server-1.6.0.tar.gz
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-1.6.0.war

# Download database connector (choose one)
# For MySQL:
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-jdbc-mysql-1.6.0.jar
# For PostgreSQL:
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-jdbc-postgresql-1.6.0.jar

# Download MySQL Connector/J (if using MySQL)
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.4.0.tar.gz
```

### Step 3.2: Build and Install guacamole-server
```bash
# Extract and build guacamole-server
cd /tmp
tar -xzf guacamole-server-1.6.0.tar.gz
cd guacamole-server-1.6.0

# Configure build
./configure --with-systemd-dir=/etc/systemd/system/

# Compile and install
make
sudo make install

# Update library cache
sudo ldconfig

# Create guacd user
sudo useradd -r -s /sbin/nologin guacd

# Enable and start guacd
sudo systemctl daemon-reload
sudo systemctl enable guacd
sudo systemctl start guacd
sudo systemctl status guacd
```

### Step 3.3: Set Up Guacamole Configuration Directory
```bash
# Create configuration directory
sudo mkdir -p /etc/guacamole/{extensions,lib}

# Create guacamole.properties
sudo tee /etc/guacamole/guacamole.properties << 'EOF'
# MySQL Configuration (adjust if using PostgreSQL)
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole
mysql-username: guacamole
mysql-password: StrongPassword123!

# PostgreSQL Configuration (uncomment if using PostgreSQL)
# postgresql-hostname: localhost
# postgresql-port: 5432
# postgresql-database: guacamole
# postgresql-username: guacamole
# postgresql-password: StrongPassword123!

# Additional settings
mysql-default-max-connections-per-user: 0
mysql-default-max-group-connections-per-user: 0
EOF

# Set permissions
sudo chown -R tomcat: /etc/guacamole
sudo chmod 600 /etc/guacamole/guacamole.properties
```

### Step 3.4: Install Database Extensions
```bash
# Copy JDBC extension (choose one based on your database)
# For MySQL:
sudo cp /tmp/guacamole-auth-jdbc-mysql-1.6.0.jar /etc/guacamole/extensions/

# For PostgreSQL:
sudo cp /tmp/guacamole-auth-jdbc-postgresql-1.6.0.jar /etc/guacamole/extensions/

# Install MySQL Connector/J (if using MySQL)
cd /tmp
tar -xzf mysql-connector-j-8.4.0.tar.gz
sudo cp mysql-connector-j-8.4.0/mysql-connector-j-8.4.0.jar /etc/guacamole/lib/

# Set permissions
sudo chown tomcat: /etc/guacamole/extensions/*
sudo chown tomcat: /etc/guacamole/lib/*
```

### Step 3.5: Deploy Guacamole Web Application
```bash
# Deploy WAR file
sudo cp /tmp/guacamole-1.6.0.war /opt/tomcat/webapps/guacamole.war

# Set ownership
sudo chown tomcat: /opt/tomcat/webapps/guacamole.war

# Set GUACAMOLE_HOME environment variable
echo 'export GUACAMOLE_HOME=/etc/guacamole' | sudo tee -a /etc/environment

# Add to Tomcat service
sudo sed -i '/Environment=JAVA_OPTS/a Environment=GUACAMOLE_HOME=/etc/guacamole' /etc/systemd/system/tomcat.service

# Reload and restart services
sudo systemctl daemon-reload
sudo systemctl restart tomcat
```

---

## Phase 4: Migrate Data from Old Server

### Step 4.1: Transfer Migration Package
```bash
# On the new server, create transfer directory
mkdir -p /tmp/migration
cd /tmp/migration

# Transfer the migration package from old server
# Option 1: Using scp (replace with your old server IP)
scp user@OLD_SERVER_IP:/tmp/guacamole-migration/guacamole-migration-*.tar.gz .

# Option 2: If you copied to external media, mount and copy
# mount /dev/sdX1 /mnt && cp /mnt/guacamole-migration-*.tar.gz .

# Extract the migration package
tar -xzf guacamole-migration-*.tar.gz
ls -la
```

### Step 4.2: Initialize Database Schema
```bash
# Download schema files
cd /tmp
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-client-1.6.0.tar.gz
tar -xzf guacamole-client-1.6.0.tar.gz

# Initialize database with 1.6.0 schema
cd guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/

# For MySQL:
cd guacamole-auth-jdbc-mysql/schema/
cat 001-create-schema.sql | mysql -u guacamole -p guacamole

# For PostgreSQL:
cd guacamole-auth-jdbc-postgresql/schema/
cat 001-create-schema.sql | sudo -u postgres psql guacamole
```

### Step 4.3: Upgrade Old Database Schema
```bash
# Navigate to upgrade scripts directory
cd /tmp/guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/

# For MySQL - run upgrade scripts in sequence:
cd guacamole-auth-jdbc-mysql/schema/upgrade/
cat upgrade-pre-1.2.0.sql | mysql -u guacamole -p guacamole
cat upgrade-pre-1.3.0.sql | mysql -u guacamole -p guacamole
cat upgrade-pre-1.4.0.sql | mysql -u guacamole -p guacamole
cat upgrade-pre-1.5.0.sql | mysql -u guacamole -p guacamole
cat upgrade-pre-1.6.0.sql | mysql -u guacamole -p guacamole

# For PostgreSQL - run upgrade scripts in sequence:
cd guacamole-auth-jdbc-postgresql/schema/upgrade/
cat upgrade-pre-1.2.0.sql | sudo -u postgres psql guacamole
cat upgrade-pre-1.3.0.sql | sudo -u postgres psql guacamole
cat upgrade-pre-1.4.0.sql | sudo -u postgres psql guacamole
cat upgrade-pre-1.5.0.sql | sudo -u postgres psql guacamole
cat upgrade-pre-1.6.0.sql | sudo -u postgres psql guacamole
```

### Step 4.4: Import Old Data
```bash
# Clear the newly created database (we'll import everything)
cd /tmp/migration

# For MySQL:
mysql -u guacamole -p -e "DROP DATABASE guacamole; CREATE DATABASE guacamole DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;"
mysql -u guacamole -p guacamole < guacamole-export.sql

# For PostgreSQL:
sudo -u postgres psql -c "DROP DATABASE guacamole; CREATE DATABASE guacamole;"
sudo -u postgres psql guacamole < guacamole-export.sql

# Now run the upgrade scripts again to bring the imported data to 1.6.0
cd /tmp/guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/

# For MySQL:
cd guacamole-auth-jdbc-mysql/schema/upgrade/
mysql -u guacamole -p guacamole < upgrade-pre-1.2.0.sql
mysql -u guacamole -p guacamole < upgrade-pre-1.3.0.sql
mysql -u guacamole -p guacamole < upgrade-pre-1.4.0.sql
mysql -u guacamole -p guacamole < upgrade-pre-1.5.0.sql
mysql -u guacamole -p guacamole < upgrade-pre-1.6.0.sql

# For PostgreSQL:
cd guacamole-auth-jdbc-postgresql/schema/upgrade/
sudo -u postgres psql guacamole < upgrade-pre-1.2.0.sql
sudo -u postgres psql guacamole < upgrade-pre-1.3.0.sql
sudo -u postgres psql guacamole < upgrade-pre-1.4.0.sql
sudo -u postgres psql guacamole < upgrade-pre-1.5.0.sql
sudo -u postgres psql guacamole < upgrade-pre-1.6.0.sql
```

### Step 4.5: Migrate Configuration and Extensions
```bash
cd /tmp/migration

# Review old configuration
cat guacamole-config/guacamole.properties

# Update configuration file with new database settings if needed
# The new server should already have the correct guacamole.properties

# Copy compatible extensions (check what was in use)
ls -la extensions/

# Install additional extensions if they were used (examples):
# Note: You'll need to download 1.6.0 versions of extensions

# If LDAP was used:
# wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-ldap-1.6.0.jar
# sudo cp guacamole-auth-ldap-1.6.0.jar /etc/guacamole/extensions/

# If TOTP was used:
# wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-totp-1.6.0.jar
# sudo cp guacamole-auth-totp-1.6.0.jar /etc/guacamole/extensions/

# Set proper permissions
sudo chown -R tomcat: /etc/guacamole/
```

---

## Phase 5: Testing and Validation

### Step 5.1: Restart Services
```bash
# Restart all services
sudo systemctl restart guacd
sudo systemctl restart tomcat

# Check service status
sudo systemctl status guacd
sudo systemctl status tomcat

# Check logs for errors
sudo journalctl -u guacd -f &
sudo tail -f /opt/tomcat/logs/catalina.out
```

### Step 5.2: Test Web Interface
```bash
# Check if Guacamole is accessible
curl -I http://localhost:8080/guacamole/

# Get server IP for external access
ip addr show | grep 'inet ' | grep -v '127.0.0.1'

echo "Access Guacamole at: http://YOUR_SERVER_IP:8080/guacamole/"
echo "Default credentials should be from your old installation"
```

### Step 5.3: Validate Migration
```bash
# Create validation script
cat > /tmp/validate-migration.sh << 'EOF'
#!/bin/bash
echo "=== Guacamole Migration Validation ==="
echo "Date: $(date)"
echo ""

# Check database connectivity
echo "=== Database Validation ==="
if command -v mysql &> /dev/null; then
    echo "User count: $(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null)"
    echo "Connection count: $(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null)"
    echo "User group count: $(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_user_group;" guacamole 2>/dev/null)"
elif command -v psql &> /dev/null; then
    echo "User count: $(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null | xargs)"
    echo "Connection count: $(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null | xargs)"
    echo "User group count: $(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_user_group;" guacamole 2>/dev/null | xargs)"
fi

echo ""
echo "=== Service Status ==="
systemctl is-active guacd
systemctl is-active tomcat

echo ""
echo "=== Network Connectivity ==="
netstat -tlnp | grep -E "(8080|4822)"

echo ""
echo "=== Extensions Installed ==="
ls -la /etc/guacamole/extensions/

echo ""
echo "=== Version Check ==="
curl -s http://localhost:8080/guacamole/ | grep -o 'Guacamole [0-9]\+\.[0-9]\+\.[0-9]\+'
EOF

chmod +x /tmp/validate-migration.sh
/tmp/validate-migration.sh
```

---

## Phase 6: Security and Optimization

### Step 6.1: Configure Firewall
```bash
# Enable UFW firewall
sudo ufw enable

# Allow SSH
sudo ufw allow ssh

# Allow Guacamole
sudo ufw allow 8080/tcp

# Allow guacd (internal communication only)
sudo ufw allow from 127.0.0.1 to any port 4822

# Check firewall status
sudo ufw status
```

### Step 6.2: Configure Reverse Proxy (Optional but Recommended)
```bash
# Install Nginx
sudo apt install -y nginx

# Create Guacamole proxy configuration
sudo tee /etc/nginx/sites-available/guacamole << 'EOF'
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    location / {
        proxy_pass http://localhost:8080/guacamole/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_cookie_path /guacamole/ /;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx

# Update firewall for Nginx
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 8080/tcp
```

### Step 6.3: Set Up SSL/TLS (Recommended)
```bash
# Install Certbot for Let's Encrypt
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate (replace with your domain)
sudo certbot --nginx -d your-domain.com

# Test automatic renewal
sudo certbot renew --dry-run
```

### Step 6.4: Create Backup Script
```bash
# Create backup script for the new server
sudo tee /usr/local/bin/guacamole-backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backup/guacamole"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/$DATE"

mkdir -p "$BACKUP_PATH"

# Backup database
if command -v mysql &> /dev/null; then
    mysqldump -u guacamole -p guacamole > "$BACKUP_PATH/guacamole-$DATE.sql"
elif command -v psql &> /dev/null; then
    sudo -u postgres pg_dump guacamole > "$BACKUP_PATH/guacamole-$DATE.sql"
fi

# Backup configuration
cp -r /etc/guacamole "$BACKUP_PATH/"

# Compress backup
tar -czf "$BACKUP_PATH.tar.gz" -C "$BACKUP_DIR" "$DATE"
rm -rf "$BACKUP_PATH"

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs rm -f

echo "Backup completed: $BACKUP_PATH.tar.gz"
EOF

sudo chmod +x /usr/local/bin/guacamole-backup.sh

# Create daily backup cron job
echo "0 2 * * * root /usr/local/bin/guacamole-backup.sh" | sudo tee -a /etc/crontab
```

---

## Phase 7: Final Steps and Cleanup

### Step 7.1: Performance Tuning
```bash
# Optimize Tomcat memory settings
sudo sed -i "s/CATALINA_OPTS='-Xms512M -Xmx1024M/CATALINA_OPTS='-Xms1024M -Xmx2048M/" /etc/systemd/system/tomcat.service

# Restart with new settings
sudo systemctl daemon-reload
sudo systemctl restart tomcat
```

### Step 7.2: Documentation
```bash
# Create final documentation
cat > /root/guacamole-migration-complete.txt << EOF
=== Guacamole Migration Completed ===
Migration Date: $(date)
Source: CentOS 7 + Guacamole 1.1.0
Target: Ubuntu 24.04 LTS + Guacamole 1.6.0

=== Access Information ===
Web Interface: http://$(hostname -I | awk '{print $1}')/guacamole/
Database: $(grep -E "(mysql|postgresql)-database" /etc/guacamole/guacamole.properties | cut -d: -f2 | xargs)

=== Service Commands ===
Restart Guacamole: sudo systemctl restart guacd tomcat
View Logs: sudo journalctl -u guacd -u tomcat -f
Backup: sudo /usr/local/bin/guacamole-backup.sh

=== Important Files ===
Configuration: /etc/guacamole/
Extensions: /etc/guacamole/extensions/
Logs: /opt/tomcat/logs/
Backups: /backup/guacamole/

=== Migration Validation ===
Run: /tmp/validate-migration.sh
EOF

echo "Migration documentation created: /root/guacamole-migration-complete.txt"
```

### Step 7.3: Cleanup
```bash
# Clean up temporary files
rm -rf /tmp/guacamole-*
rm -rf /tmp/migration
rm -rf /tmp/mysql-connector-*

# Remove old packages cache
sudo apt autoremove -y
sudo apt autoclean
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Database Connection Issues
```bash
# Test database connectivity
mysql -u guacamole -p -e "SELECT COUNT(*) FROM guacamole_user;" guacamole
# or
sudo -u postgres psql -c "SELECT COUNT(*) FROM guacamole_user;" guacamole

# Check configuration
cat /etc/guacamole/guacamole.properties | grep -E "(mysql|postgresql)"
```

#### 2. Service Start Issues
```bash
# Check detailed service status
sudo journalctl -u guacd --no-pager
sudo journalctl -u tomcat --no-pager

# Check if ports are in use
sudo netstat -tlnp | grep -E "(8080|4822)"
```

#### 3. Web Interface Not Loading
```bash
# Check Tomcat logs
sudo tail -f /opt/tomcat/logs/catalina.out
sudo tail -f /opt/tomcat/logs/localhost.*.log

# Verify WAR deployment
ls -la /opt/tomcat/webapps/
```

#### 4. Authentication Issues
```bash
# Reset admin password if needed (run in database)
# For MySQL:
mysql -u guacamole -p guacamole -e "UPDATE guacamole_user SET password_hash = UNHEX(SHA2(CONCAT('password', HEX(password_salt)), 256)) WHERE username = 'guacadmin';"

# For PostgreSQL:
sudo -u postgres psql guacamole -c "UPDATE guacamole_user SET password_hash = decode(sha256(concat(password_salt, 'password')::bytea), 'hex') WHERE username = 'guacadmin';"
```

---

## Migration Checklist

- [ ] Phase 1: Old server documentation and backup complete
- [ ] Phase 2: New server prepared with Ubuntu 24.04
- [ ] Phase 3: Guacamole 1.6.0 installed successfully
- [ ] Phase 4: Data migration completed
- [ ] Phase 5: Testing and validation passed
- [ ] Phase 6: Security and optimization configured
- [ ] Phase 7: Documentation and cleanup completed
- [ ] Troubleshooting guide reviewed
- [ ] Old server safely decommissioned (after validation period)

---

## Post-Migration Notes

1. **Monitor the new server** for 24-48 hours to ensure stability
2. **Test all connection types** (RDP, VNC, SSH) that were previously configured
3. **Verify user authentication** and permissions
4. **Test any integrated systems** (LDAP, SAML, etc.)
5. **Update DNS/firewall rules** to point to the new server
6. **Schedule regular backups** using the provided script
7. **Keep the old server running** for a few days as a fallback option

The migration is now complete. Your new Ubuntu server should be running Apache Guacamole 1.6.0 with all data and configurations migrated from the old CentOS 7 server.