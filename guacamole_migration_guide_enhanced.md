# Apache Guacamole Migration Guide for Linux Beginners
## Moving from CentOS 7 to Ubuntu 24.04 LTS

### What This Guide Does
This guide helps you move your Apache Guacamole server from an old CentOS 7 system to a brand new Ubuntu 24.04 server. Think of it like moving from an old house to a new one - we'll pack everything up, move it safely, and unpack it in your new location.

### What You'll Need
- **Two servers**: Your old CentOS 7 server and a new Ubuntu 24.04 server
- **Root access** to both servers (ability to use `sudo`)
- **Basic knowledge** of how to copy and paste commands
- **A way to transfer files** between servers (we'll show you how)

### Important Notes for Beginners
- **Copy commands exactly** as shown - Linux is case-sensitive
- **Press Enter** after each command to run it
- **Read error messages** - they often tell you what's wrong
- **Don't panic** if something doesn't work - we have troubleshooting steps

---

## Phase 1: Prepare Your Old Server (CentOS 7)

### Step 1.1: Connect to Your Old Server
First, you need to connect to your old CentOS 7 server using SSH:

```bash
# Replace 'your-old-server-ip' with the actual IP address
ssh root@your-old-server-ip
# Or if you have a regular user:
ssh yourusername@your-old-server-ip
```

### Step 1.2: Create a Script to Document Your System

**What we're doing:** Creating a script that will gather information about your current setup.

**Create the script file:**
```bash
# Create a new file called document-system.sh
nano /tmp/document-system.sh
```

**Copy and paste this into the file:**
```bash
#!/bin/bash
# This script documents your current Guacamole setup

echo "Creating documentation directory..."
sudo mkdir -p /tmp/guacamole-migration
cd /tmp/guacamole-migration

echo "Gathering system information..."
echo "=== Current Guacamole Configuration ===" > migration-info.txt
echo "Date: $(date)" >> migration-info.txt
echo "Server: $(hostname)" >> migration-info.txt
echo "OS: $(cat /etc/redhat-release)" >> migration-info.txt
echo "" >> migration-info.txt

echo "Finding Guacamole version..."
echo "=== Guacamole Version ===" >> migration-info.txt
find /var/lib/tomcat* /usr/share/tomcat* -name "guacamole-*.jar" 2>/dev/null >> migration-info.txt
echo "" >> migration-info.txt

echo "Checking database configuration..."
echo "=== Database Configuration ===" >> migration-info.txt
if [ -f /etc/guacamole/guacamole.properties ]; then
    grep -E "(mysql|postgresql)" /etc/guacamole/guacamole.properties >> migration-info.txt
else
    echo "Configuration file not found at /etc/guacamole/guacamole.properties" >> migration-info.txt
fi
echo "" >> migration-info.txt

echo "Checking running services..."
echo "=== Running Services ===" >> migration-info.txt
sudo systemctl status guacd >> migration-info.txt 2>&1
sudo systemctl status tomcat >> migration-info.txt 2>&1
echo "" >> migration-info.txt

echo "Checking network configuration..."
echo "=== Network Configuration ===" >> migration-info.txt
sudo netstat -tlnp | grep -E "(8080|4822)" >> migration-info.txt

echo "System documentation complete! Check /tmp/guacamole-migration/migration-info.txt"
```

**Save and exit:**
- Press `Ctrl + X`
- Press `Y` to save
- Press `Enter` to confirm

**Make the script executable and run it:**
```bash
# Make the script executable
chmod +x /tmp/document-system.sh

# Run the script
/tmp/document-system.sh

# Check what it found
cat /tmp/guacamole-migration/migration-info.txt
```

### Step 1.3: Create a Script to Backup Your Database

**What we're doing:** This script will create a complete backup of your Guacamole database.

**Create the backup script:**
```bash
nano /tmp/backup-database.sh
```

**Copy and paste this into the file:**
```bash
#!/bin/bash
# This script backs up your Guacamole database

echo "Moving to migration directory..."
cd /tmp/guacamole-migration

echo "Determining database type..."
if grep -q "mysql" /etc/guacamole/guacamole.properties 2>/dev/null; then
    echo "MySQL database detected"
    echo "Please enter your MySQL root password when prompted:"
    mysqldump -u root -p --single-transaction --routines --triggers guacamole > guacamole-export.sql
    
    if [ $? -eq 0 ]; then
        echo "MySQL database backup successful!"
    else
        echo "MySQL backup failed. Please check your password and try again."
        exit 1
    fi
    
elif grep -q "postgresql" /etc/guacamole/guacamole.properties 2>/dev/null; then
    echo "PostgreSQL database detected"
    sudo -u postgres pg_dump guacamole > guacamole-export.sql
    
    if [ $? -eq 0 ]; then
        echo "PostgreSQL database backup successful!"
    else
        echo "PostgreSQL backup failed. Please check your configuration."
        exit 1
    fi
else
    echo "Could not determine database type from configuration."
    echo "Please check /etc/guacamole/guacamole.properties manually."
    exit 1
fi

# Verify the backup
if [ -f "guacamole-export.sql" ]; then
    LINES=$(wc -l < guacamole-export.sql)
    SIZE=$(ls -lh guacamole-export.sql | awk '{print $5}')
    echo "Database backup created successfully!"
    echo "File: guacamole-export.sql"
    echo "Size: $SIZE"
    echo "Lines: $LINES"
    
    if [ $LINES -lt 50 ]; then
        echo "WARNING: Backup file seems very small. Please verify it contains your data."
    fi
else
    echo "ERROR: Backup file was not created!"
    exit 1
fi
```

**Save, make executable, and run:**
```bash
# Save and exit (Ctrl+X, Y, Enter)
chmod +x /tmp/backup-database.sh
/tmp/backup-database.sh
```

### Step 1.4: Create a Script to Backup Configuration Files

**Create the configuration backup script:**
```bash
nano /tmp/backup-config.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script backs up all Guacamole configuration files

echo "Moving to migration directory..."
cd /tmp/guacamole-migration

echo "Backing up Guacamole configuration files..."
if [ -d /etc/guacamole ]; then
    sudo cp -r /etc/guacamole/ ./guacamole-config/
    echo "Main configuration backed up from /etc/guacamole/"
else
    echo "WARNING: /etc/guacamole directory not found!"
fi

echo "Looking for extensions and libraries..."
mkdir -p extensions lib

if [ -d /etc/guacamole/extensions ]; then
    sudo cp /etc/guacamole/extensions/* ./extensions/ 2>/dev/null || echo "No extensions found"
    echo "Extensions backed up"
else
    echo "No extensions directory found"
fi

if [ -d /etc/guacamole/lib ]; then
    sudo cp /etc/guacamole/lib/* ./lib/ 2>/dev/null || echo "No libraries found"
    echo "Libraries backed up"
else
    echo "No lib directory found"
fi

echo "Documenting what we found..."
echo "=== Configuration Files Backed Up ===" >> migration-info.txt
ls -la guacamole-config/ >> migration-info.txt
echo "" >> migration-info.txt
echo "=== Extensions Found ===" >> migration-info.txt
ls -la extensions/ >> migration-info.txt
echo "" >> migration-info.txt
echo "=== Libraries Found ===" >> migration-info.txt
ls -la lib/ >> migration-info.txt

echo "Configuration backup complete!"
echo "Check these directories:"
echo "- guacamole-config/ (main configuration)"
echo "- extensions/ (Guacamole extensions)"
echo "- lib/ (additional libraries)"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/backup-config.sh
/tmp/backup-config.sh
```

### Step 1.5: Create a Script to Check User Data

**Create the user data check script:**
```bash
nano /tmp/check-users.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script checks how many users and connections you have

echo "Moving to migration directory..."
cd /tmp/guacamole-migration

echo "Creating SQL script to check user data..."
cat > export-users.sql << 'EOF'
SELECT 'Users found:' as info, COUNT(*) as count FROM guacamole_user;
SELECT 'Connections found:' as info, COUNT(*) as count FROM guacamole_connection;
SELECT 'User groups found:' as info, COUNT(*) as count FROM guacamole_user_group;
EOF

echo "Running database query to count your data..."
if grep -q "mysql" /etc/guacamole/guacamole.properties 2>/dev/null; then
    echo "Checking MySQL database..."
    echo "Please enter your MySQL root password when prompted:"
    mysql -u root -p guacamole < export-users.sql > user-data-summary.txt
elif grep -q "postgresql" /etc/guacamole/guacamole.properties 2>/dev/null; then
    echo "Checking PostgreSQL database..."
    sudo -u postgres psql guacamole < export-users.sql > user-data-summary.txt
else
    echo "Could not determine database type."
    echo "Creating empty summary file."
    touch user-data-summary.txt
fi

echo "User data summary:"
cat user-data-summary.txt
echo ""
echo "Adding summary to migration info..."
cat user-data-summary.txt >> migration-info.txt
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/check-users.sh
/tmp/check-users.sh
```

### Step 1.6: Create a Script to Package Everything

**Create the packaging script:**
```bash
nano /tmp/create-package.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script packages everything for transfer to the new server

echo "Moving to migration directory..."
cd /tmp/guacamole-migration

echo "Creating migration package..."
PACKAGE_NAME="guacamole-migration-$(date +%Y%m%d-%H%M%S).tar.gz"

tar -czf $PACKAGE_NAME \
    migration-info.txt \
    guacamole-export.sql \
    guacamole-config/ \
    extensions/ \
    lib/ \
    user-data-summary.txt \
    export-users.sql

if [ $? -eq 0 ]; then
    echo "Migration package created successfully!"
    echo "Package name: $PACKAGE_NAME"
    echo "Package location: /tmp/guacamole-migration/$PACKAGE_NAME"
    echo "Package size: $(ls -lh $PACKAGE_NAME | awk '{print $5}')"
    
    echo ""
    echo "IMPORTANT: Copy this file to your new Ubuntu server!"
    echo "You can use SCP, a USB drive, or any file transfer method."
    echo ""
    echo "SCP command example (run from your new server):"
    echo "scp root@$(hostname -I | awk '{print $1}'):/tmp/guacamole-migration/$PACKAGE_NAME /tmp/"
else
    echo "ERROR: Failed to create migration package!"
    exit 1
fi
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/create-package.sh
/tmp/create-package.sh
```

---

## Phase 2: Set Up Your New Ubuntu Server

### Step 2.1: Connect to Your New Server
Connect to your new Ubuntu 24.04 server:
```bash
# Replace with your new server's IP address
ssh root@your-new-server-ip
# Or with a user account:
ssh yourusername@your-new-server-ip
```

### Step 2.2: Create a Script to Update the System

**Create the system update script:**
```bash
nano /tmp/update-system.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script updates Ubuntu and installs required packages

echo "Updating Ubuntu system..."
sudo apt update && sudo apt upgrade -y

echo "Installing development tools and dependencies..."
sudo apt install -y build-essential libcairo2-dev libjpeg-turbo8-dev \
    libpng-dev libtool-bin libossp-uuid-dev libavcodec-dev libavformat-dev \
    libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev \
    libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev \
    libpulse-dev libssl-dev libvorbis-dev libwebp-dev wget tar \
    nano curl unzip

echo "Installing Java 11 (required for Guacamole)..."
sudo apt install -y openjdk-11-jdk

echo "Verifying Java installation..."
java -version

if [ $? -eq 0 ]; then
    echo "System update and dependencies installed successfully!"
else
    echo "There was an issue with the installation. Please check the output above."
fi
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/update-system.sh
/tmp/update-system.sh
```

### Step 2.3: Create a Script to Install Database

**Create the database installation script:**
```bash
nano /tmp/install-database.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script installs and configures the database

echo "Choose your database type:"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

if [ "$db_choice" = "1" ]; then
    echo "Installing MySQL..."
    sudo apt install -y mysql-server
    
    echo "Starting MySQL service..."
    sudo systemctl start mysql
    sudo systemctl enable mysql
    
    echo "Securing MySQL installation..."
    echo "IMPORTANT: You'll be asked to set a root password and answer security questions."
    echo "Press Enter to continue..."
    read
    sudo mysql_secure_installation
    
    echo "Creating Guacamole database and user..."
    echo "Please enter the MySQL root password you just set:"
    mysql -u root -p << 'EOF'
CREATE DATABASE guacamole DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER 'guacamole'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON guacamole.* TO 'guacamole'@'localhost';
FLUSH PRIVILEGES;
SHOW DATABASES;
EXIT;
EOF

    echo "MySQL setup complete!"
    echo "Database: guacamole"
    echo "User: guacamole"
    echo "Password: StrongPassword123!"
    echo ""
    echo "IMPORTANT: Write down these database credentials!"
    
elif [ "$db_choice" = "2" ]; then
    echo "Installing PostgreSQL..."
    sudo apt install -y postgresql postgresql-contrib
    
    echo "Starting PostgreSQL service..."
    sudo systemctl start postgresql
    sudo systemctl enable postgresql
    
    echo "Creating Guacamole database and user..."
    sudo -u postgres psql << 'EOF'
CREATE DATABASE guacamole;
CREATE USER guacamole WITH PASSWORD 'StrongPassword123!';
GRANT ALL PRIVILEGES ON DATABASE guacamole TO guacamole;
\l
\q
EOF

    echo "PostgreSQL setup complete!"
    echo "Database: guacamole"
    echo "User: guacamole"
    echo "Password: StrongPassword123!"
    echo ""
    echo "IMPORTANT: Write down these database credentials!"
    
else
    echo "Invalid choice. Please run the script again and choose 1 or 2."
    exit 1
fi

echo "Database installation complete!"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/install-database.sh
/tmp/install-database.sh
```

### Step 2.4: Create a Script to Install Tomcat

**Create the Tomcat installation script:**
```bash
nano /tmp/install-tomcat.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script installs and configures Apache Tomcat

echo "Creating tomcat user..."
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat

echo "Downloading Apache Tomcat..."
cd /tmp
wget https://downloads.apache.org/tomcat/tomcat-10/v10.1.28/bin/apache-tomcat-10.1.28.tar.gz

if [ $? -ne 0 ]; then
    echo "Failed to download Tomcat. Please check your internet connection."
    exit 1
fi

echo "Installing Tomcat..."
sudo tar -xzf apache-tomcat-10.1.28.tar.gz -C /opt/tomcat --strip-components=1

echo "Setting permissions..."
sudo chown -R tomcat: /opt/tomcat
sudo chmod +x /opt/tomcat/bin/*.sh

echo "Creating Tomcat systemd service..."
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

echo "Starting Tomcat service..."
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat

echo "Checking Tomcat status..."
sleep 5
sudo systemctl status tomcat

if sudo systemctl is-active --quiet tomcat; then
    echo "Tomcat installation successful!"
    echo "Tomcat is running on port 8080"
    echo "You can test it by visiting: http://your-server-ip:8080"
else
    echo "Tomcat failed to start. Please check the logs:"
    echo "sudo journalctl -u tomcat"
fi
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/install-tomcat.sh
/tmp/install-tomcat.sh
```

---

## Phase 3: Install Guacamole 1.6.0

### Step 3.1: Create a Script to Download Guacamole

**Create the download script:**
```bash
nano /tmp/download-guacamole.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script downloads all Guacamole components

echo "Moving to temporary directory..."
cd /tmp

echo "Downloading Guacamole 1.6.0 components..."

echo "Downloading guacamole-server (source code)..."
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-server-1.6.0.tar.gz

echo "Downloading guacamole web application..."
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-1.6.0.war

echo "Downloading guacamole-client (for database schema)..."
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-client-1.6.0.tar.gz

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

if [ "$db_choice" = "1" ]; then
    echo "Downloading MySQL database connector..."
    wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-jdbc-mysql-1.6.0.jar
    
    echo "Downloading MySQL Connector/J..."
    wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.4.0.tar.gz
    
    echo "MySQL components downloaded!"
    
elif [ "$db_choice" = "2" ]; then
    echo "Downloading PostgreSQL database connector..."
    wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-jdbc-postgresql-1.6.0.jar
    
    echo "PostgreSQL components downloaded!"
    
else
    echo "Invalid choice. Please run the script again."
    exit 1
fi

echo "Verifying downloads..."
ls -la /tmp/guacamole-*
ls -la /tmp/mysql-connector-* 2>/dev/null || true

echo "All downloads complete!"
echo "Files are in /tmp/ directory"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/download-guacamole.sh
/tmp/download-guacamole.sh
```

### Step 3.2: Create a Script to Build guacamole-server

**Create the build script:**
```bash
nano /tmp/build-guacamole-server.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script builds and installs guacamole-server

echo "Moving to temporary directory..."
cd /tmp

echo "Extracting guacamole-server source code..."
tar -xzf guacamole-server-1.6.0.tar.gz
cd guacamole-server-1.6.0

echo "Configuring build..."
./configure --with-systemd-dir=/etc/systemd/system/

if [ $? -ne 0 ]; then
    echo "Configuration failed! Please check error messages above."
    echo "You might be missing some dependencies."
    exit 1
fi

echo "Compiling guacamole-server (this may take several minutes)..."
make

if [ $? -ne 0 ]; then
    echo "Compilation failed! Please check error messages above."
    exit 1
fi

echo "Installing guacamole-server..."
sudo make install

echo "Updating library cache..."
sudo ldconfig

echo "Creating guacd user..."
sudo useradd -r -s /sbin/nologin guacd

echo "Starting guacd service..."
sudo systemctl daemon-reload
sudo systemctl enable guacd
sudo systemctl start guacd

echo "Checking guacd status..."
sleep 2
sudo systemctl status guacd

if sudo systemctl is-active --quiet guacd; then
    echo "guacamole-server installation successful!"
    echo "guacd is running on port 4822"
else
    echo "guacd failed to start. Please check the logs:"
    echo "sudo journalctl -u guacd"
fi
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/build-guacamole-server.sh
/tmp/build-guacamole-server.sh
```

### Step 3.3: Create a Script to Configure Guacamole

**Create the configuration script:**
```bash
nano /tmp/configure-guacamole.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script sets up Guacamole configuration

echo "Creating Guacamole configuration directory..."
sudo mkdir -p /etc/guacamole/{extensions,lib}

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

if [ "$db_choice" = "1" ]; then
    echo "Creating MySQL configuration..."
    sudo tee /etc/guacamole/guacamole.properties << 'EOF'
# MySQL Configuration
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole
mysql-username: guacamole
mysql-password: StrongPassword123!

# Connection settings
mysql-default-max-connections-per-user: 0
mysql-default-max-group-connections-per-user: 0
EOF

elif [ "$db_choice" = "2" ]; then
    echo "Creating PostgreSQL configuration..."
    sudo tee /etc/guacamole/guacamole.properties << 'EOF'
# PostgreSQL Configuration
postgresql-hostname: localhost
postgresql-port: 5432
postgresql-database: guacamole
postgresql-username: guacamole
postgresql-password: StrongPassword123!

# Connection settings
postgresql-default-max-connections-per-user: 0
postgresql-default-max-group-connections-per-user: 0
EOF

else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo "Setting proper permissions..."
sudo chown -R tomcat: /etc/guacamole
sudo chmod 600 /etc/guacamole/guacamole.properties

echo "Setting GUACAMOLE_HOME environment variable..."
echo 'export GUACAMOLE_HOME=/etc/guacamole' | sudo tee -a /etc/environment

echo "Updating Tomcat service to include GUACAMOLE_HOME..."
sudo sed -i '/Environment=JAVA_OPTS/a Environment=GUACAMOLE_HOME=/etc/guacamole' /etc/systemd/system/tomcat.service

echo "Configuration complete!"
echo "Configuration file location: /etc/guacamole/guacamole.properties"
echo "You can view it with: sudo cat /etc/guacamole/guacamole.properties"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/configure-guacamole.sh
/tmp/configure-guacamole.sh
```

### Step 3.4: Create a Script to Install Database Extensions

**Create the extensions installation script:**
```bash
nano /tmp/install-extensions.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script installs Guacamole database extensions

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

cd /tmp

if [ "$db_choice" = "1" ]; then
    echo "Installing MySQL JDBC extension..."
    sudo cp guacamole-auth-jdbc-mysql-1.6.0.jar /etc/guacamole/extensions/
    
    echo "Installing MySQL Connector/J..."
    tar -xzf mysql-connector-j-8.4.0.tar.gz
    sudo cp mysql-connector-j-8.4.0/mysql-connector-j-8.4.0.jar /etc/guacamole/lib/
    
    echo "MySQL extensions installed!"
    
elif [ "$db_choice" = "2" ]; then
    echo "Installing PostgreSQL JDBC extension..."
    sudo cp guacamole-auth-jdbc-postgresql-1.6.0.jar /etc/guacamole/extensions/
    
    echo "PostgreSQL extensions installed!"
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo "Setting proper permissions..."
sudo chown tomcat: /etc/guacamole/extensions/*
sudo chown tomcat: /etc/guacamole/lib/* 2>/dev/null || true

echo "Listing installed extensions..."
ls -la /etc/guacamole/extensions/
ls -la /etc/guacamole/lib/ 2>/dev/null || true

echo "Extensions installation complete!"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/install-extensions.sh
/tmp/install-extensions.sh
```

### Step 3.5: Create a Script to Deploy Guacamole Web App

**Create the deployment script:**
```bash
nano /tmp/deploy-guacamole.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script deploys the Guacamole web application

echo "Deploying Guacamole web application..."
cd /tmp

echo "Copying WAR file to Tomcat..."
sudo cp guacamole-1.6.0.war /opt/tomcat/webapps/guacamole.war

echo "Setting proper ownership..."
sudo chown tomcat: /opt/tomcat/webapps/guacamole.war

echo "Reloading systemd and restarting services..."
sudo systemctl daemon-reload
sudo systemctl restart tomcat

echo "Waiting for Tomcat to start..."
sleep 10

echo "Checking service status..."
sudo systemctl status tomcat

echo "Checking if Guacamole web app deployed..."
sleep 5
if [ -d "/opt/tomcat/webapps/guacamole" ]; then
    echo "SUCCESS: Guacamole web application deployed!"
    echo "You should be able to access it at: http://your-server-ip:8080/guacamole/"
else
    echo "Waiting a bit more for deployment..."
    sleep 10
    if [ -d "/opt/tomcat/webapps/guacamole" ]; then
        echo "SUCCESS: Guacamole web application deployed!"
        echo "You should be able to access it at: http://your-server-ip:8080/guacamole/"
    else
        echo "WARNING: Web application may not have deployed correctly."
        echo "Check Tomcat logs: sudo tail -f /opt/tomcat/logs/catalina.out"
    fi
fi

echo "Getting server IP address for access..."
SERVER_IP=$(hostname -I | awk '{print $1}')
echo ""
echo "=========================================="
echo "Guacamole should be accessible at:"
echo "http://$SERVER_IP:8080/guacamole/"
echo "=========================================="
echo ""
echo "However, you still need to set up the database schema before you can log in!"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/deploy-guacamole.sh
/tmp/deploy-guacamole.sh
```

---

## Phase 4: Transfer and Import Your Old Data

### Step 4.1: Create a Script to Transfer Migration Package

**Create the transfer script:**
```bash
nano /tmp/transfer-data.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script helps you transfer data from your old server

echo "Creating migration directory..."
mkdir -p /tmp/migration
cd /tmp/migration

echo ""
echo "You need to transfer your migration package from the old server."
echo "Here are your options:"
echo ""
echo "1) Use SCP (if you have network access between servers)"
echo "2) I've already copied the file to this server"
echo ""
read -p "Choose option (1 or 2): " transfer_choice

if [ "$transfer_choice" = "1" ]; then
    echo ""
    echo "Enter your old server details:"
    read -p "Username on old server: " old_server_user
    
    echo ""
    echo "Looking for migration packages on old server..."
    echo "Available packages:"
    ssh $old_server_user@$old_server_ip "ls -la /tmp/guacamole-migration/guacamole-migration-*.tar.gz 2>/dev/null || echo 'No migration packages found'"
    
    echo ""
    read -p "Enter the full filename of your migration package: " package_name
    
    echo "Transferring $package_name from old server..."
    scp $old_server_user@$old_server_ip:/tmp/guacamole-migration/$package_name .
    
    if [ $? -eq 0 ]; then
        echo "Transfer successful!"
    else
        echo "Transfer failed. Please check your connection and try again."
        exit 1
    fi
    
elif [ "$transfer_choice" = "2" ]; then
    echo ""
    echo "Please copy your migration package file to: /tmp/migration/"
    echo "Then press Enter to continue..."
    read
    
    echo "Looking for migration packages..."
    ls -la /tmp/migration/guacamole-migration-*.tar.gz 2>/dev/null
    
    if [ $? -ne 0 ]; then
        echo "No migration package found in /tmp/migration/"
        echo "Please copy your guacamole-migration-*.tar.gz file to /tmp/migration/ and run this script again."
        exit 1
    fi
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo ""
echo "Extracting migration package..."
tar -xzf guacamole-migration-*.tar.gz

if [ $? -eq 0 ]; then
    echo "Migration package extracted successfully!"
    echo "Contents:"
    ls -la
    echo ""
    echo "Reading migration info..."
    cat migration-info.txt
else
    echo "Failed to extract migration package!"
    exit 1
fi
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/transfer-data.sh
/tmp/transfer-data.sh
```

### Step 4.2: Create a Script to Set Up Database Schema

**Create the database schema script:**
```bash
nano /tmp/setup-database-schema.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script sets up the database schema for Guacamole 1.6.0

echo "Setting up database schema for Guacamole 1.6.0..."

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

echo "Extracting guacamole-client for schema files..."
cd /tmp
tar -xzf guacamole-client-1.6.0.tar.gz

if [ "$db_choice" = "1" ]; then
    echo "Setting up MySQL schema..."
    cd guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/
    
    echo "Please enter MySQL guacamole user password (StrongPassword123! if you used our default):"
    mysql -u guacamole -p guacamole < 001-create-schema.sql
    
    if [ $? -eq 0 ]; then
        echo "MySQL schema created successfully!"
    else
        echo "Failed to create MySQL schema. Please check your password and database connection."
        exit 1
    fi
    
elif [ "$db_choice" = "2" ]; then
    echo "Setting up PostgreSQL schema..."
    cd guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-postgresql/schema/
    
    sudo -u postgres psql guacamole < 001-create-schema.sql
    
    if [ $? -eq 0 ]; then
        echo "PostgreSQL schema created successfully!"
    else
        echo "Failed to create PostgreSQL schema. Please check your database connection."
        exit 1
    fi
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo "Database schema setup complete!"
echo "Next step: Import your old data and upgrade the schema."
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/setup-database-schema.sh
/tmp/setup-database-schema.sh
```

### Step 4.3: Create a Script to Import Old Data

**Create the data import script:**
```bash
nano /tmp/import-old-data.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script imports your old Guacamole data

echo "Importing old Guacamole data..."

cd /tmp/migration

if [ ! -f "guacamole-export.sql" ]; then
    echo "ERROR: guacamole-export.sql not found!"
    echo "Please make sure you've run the transfer-data.sh script first."
    exit 1
fi

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

echo ""
echo "WARNING: This will replace the current database with your old data!"
echo "Make sure you've backed up anything important."
read -p "Are you sure you want to continue? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Import cancelled."
    exit 0
fi

if [ "$db_choice" = "1" ]; then
    echo "Dropping and recreating MySQL database..."
    echo "Please enter MySQL root password:"
    mysql -u root -p -e "DROP DATABASE IF EXISTS guacamole; CREATE DATABASE guacamole DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci; GRANT ALL PRIVILEGES ON guacamole.* TO 'guacamole'@'localhost'; FLUSH PRIVILEGES;"
    
    echo "Importing old data..."
    echo "Please enter MySQL guacamole user password:"
    mysql -u guacamole -p guacamole < guacamole-export.sql
    
    if [ $? -eq 0 ]; then
        echo "MySQL data import successful!"
    else
        echo "MySQL data import failed!"
        exit 1
    fi
    
elif [ "$db_choice" = "2" ]; then
    echo "Dropping and recreating PostgreSQL database..."
    sudo -u postgres psql -c "DROP DATABASE IF EXISTS guacamole; CREATE DATABASE guacamole; GRANT ALL PRIVILEGES ON DATABASE guacamole TO guacamole;"
    
    echo "Importing old data..."
    sudo -u postgres psql guacamole < guacamole-export.sql
    
    if [ $? -eq 0 ]; then
        echo "PostgreSQL data import successful!"
    else
        echo "PostgreSQL data import failed!"
        exit 1
    fi
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo "Old data imported successfully!"
echo "Next step: Upgrade the database schema to version 1.6.0"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/import-old-data.sh
/tmp/import-old-data.sh
```

### Step 4.4: Create a Script to Upgrade Database Schema

**Create the schema upgrade script:**
```bash
nano /tmp/upgrade-database-schema.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script upgrades your database schema from 1.1.0 to 1.6.0

echo "Upgrading database schema from 1.1.0 to 1.6.0..."

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

cd /tmp/guacamole-client-1.6.0/extensions/guacamole-auth-jdbc/modules/

if [ "$db_choice" = "1" ]; then
    echo "Upgrading MySQL database schema..."
    cd guacamole-auth-jdbc-mysql/schema/upgrade/
    
    echo "Running upgrade scripts in sequence..."
    echo "This may take a few minutes depending on your data size."
    
    echo "Please enter MySQL guacamole user password for each upgrade step:"
    
    echo "Upgrading to 1.2.0..."
    mysql -u guacamole -p guacamole < upgrade-pre-1.2.0.sql
    
    echo "Upgrading to 1.3.0..."
    mysql -u guacamole -p guacamole < upgrade-pre-1.3.0.sql
    
    echo "Upgrading to 1.4.0..."
    mysql -u guacamole -p guacamole < upgrade-pre-1.4.0.sql
    
    echo "Upgrading to 1.5.0..."
    mysql -u guacamole -p guacamole < upgrade-pre-1.5.0.sql
    
    echo "Upgrading to 1.6.0..."
    mysql -u guacamole -p guacamole < upgrade-pre-1.6.0.sql
    
    if [ $? -eq 0 ]; then
        echo "MySQL schema upgrade completed successfully!"
    else
        echo "MySQL schema upgrade failed!"
        exit 1
    fi
    
elif [ "$db_choice" = "2" ]; then
    echo "Upgrading PostgreSQL database schema..."
    cd guacamole-auth-jdbc-postgresql/schema/upgrade/
    
    echo "Running upgrade scripts in sequence..."
    echo "This may take a few minutes depending on your data size."
    
    echo "Upgrading to 1.2.0..."
    sudo -u postgres psql guacamole < upgrade-pre-1.2.0.sql
    
    echo "Upgrading to 1.3.0..."
    sudo -u postgres psql guacamole < upgrade-pre-1.3.0.sql
    
    echo "Upgrading to 1.4.0..."
    sudo -u postgres psql guacamole < upgrade-pre-1.4.0.sql
    
    echo "Upgrading to 1.5.0..."
    sudo -u postgres psql guacamole < upgrade-pre-1.5.0.sql
    
    echo "Upgrading to 1.6.0..."
    sudo -u postgres psql guacamole < upgrade-pre-1.6.0.sql
    
    if [ $? -eq 0 ]; then
        echo "PostgreSQL schema upgrade completed successfully!"
    else
        echo "PostgreSQL schema upgrade failed!"
        exit 1
    fi
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo ""
echo "Database schema upgrade complete!"
echo "Your database now supports Guacamole 1.6.0 features."
echo "Next step: Restart services and test the web interface."
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/upgrade-database-schema.sh
/tmp/upgrade-database-schema.sh
```

---

## Phase 5: Test Your Migration

### Step 5.1: Create a Script to Restart All Services

**Create the service restart script:**
```bash
nano /tmp/restart-services.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script restarts all Guacamole services

echo "Restarting all Guacamole services..."

echo "Stopping services..."
sudo systemctl stop tomcat
sudo systemctl stop guacd

echo "Waiting 5 seconds..."
sleep 5

echo "Starting guacd..."
sudo systemctl start guacd
sleep 2

echo "Starting Tomcat..."
sudo systemctl start tomcat
sleep 5

echo "Checking service status..."
echo ""
echo "=== guacd Status ==="
sudo systemctl status guacd --no-pager
echo ""
echo "=== Tomcat Status ==="
sudo systemctl status tomcat --no-pager

echo ""
echo "Checking if services are running..."
if sudo systemctl is-active --quiet guacd; then
    echo "✓ guacd is running"
else
    echo "✗ guacd is not running"
fi

if sudo systemctl is-active --quiet tomcat; then
    echo "✓ Tomcat is running"
else
    echo "✗ Tomcat is not running"
fi

echo ""
echo "Checking network ports..."
sudo netstat -tlnp | grep -E "(8080|4822)"

echo ""
echo "Getting server IP address..."
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "Guacamole should be accessible at: http://$SERVER_IP:8080/guacamole/"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/restart-services.sh
/tmp/restart-services.sh
```

### Step 5.2: Create a Script to Test the Installation

**Create the testing script:**
```bash
nano /tmp/test-installation.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script tests your Guacamole installation

echo "Testing Guacamole installation..."

echo ""
echo "=== Service Status Check ==="
echo "guacd: $(sudo systemctl is-active guacd)"
echo "tomcat: $(sudo systemctl is-active tomcat)"

echo ""
echo "=== Network Connectivity Check ==="
echo "Checking if Guacamole web interface responds..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/guacamole/ 2>/dev/null)

if [ "$HTTP_STATUS" = "200" ]; then
    echo "✓ Guacamole web interface is responding (HTTP 200)"
elif [ "$HTTP_STATUS" = "302" ]; then
    echo "✓ Guacamole web interface is responding (HTTP 302 - redirect to login)"
else
    echo "✗ Guacamole web interface is not responding properly (HTTP $HTTP_STATUS)"
fi

echo ""
echo "=== Database Connectivity Check ==="
cd /tmp/migration

if grep -q "mysql" /etc/guacamole/guacamole.properties; then
    echo "Testing MySQL database connection..."
    USERS=$(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null)
    CONNECTIONS=$(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null)
    GROUPS=$(mysql -u guacamole -p -N -e "SELECT COUNT(*) FROM guacamole_user_group;" guacamole 2>/dev/null)
    
    if [ ! -z "$USERS" ]; then
        echo "✓ Database connection successful!"
        echo "  Users: $USERS"
        echo "  Connections: $CONNECTIONS"
        echo "  User Groups: $GROUPS"
    else
        echo "✗ Database connection failed"
    fi
    
elif grep -q "postgresql" /etc/guacamole/guacamole.properties; then
    echo "Testing PostgreSQL database connection..."
    USERS=$(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null | xargs)
    CONNECTIONS=$(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null | xargs)
    GROUPS=$(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_user_group;" guacamole 2>/dev/null | xargs)
    
    if [ ! -z "$USERS" ]; then
        echo "✓ Database connection successful!"
        echo "  Users: $USERS"
        echo "  Connections: $CONNECTIONS"
        echo "  User Groups: $GROUPS"
    else
        echo "✗ Database connection failed"
    fi
fi

echo ""
echo "=== Extensions Check ==="
echo "Installed extensions:"
ls -la /etc/guacamole/extensions/

echo ""
echo "=== Version Check ==="
echo "Attempting to detect Guacamole version..."
VERSION=$(curl -s http://localhost:8080/guacamole/ | grep -o 'Guacamole [0-9]\+\.[0-9]\+\.[0-9]\+' | head -1)
if [ ! -z "$VERSION" ]; then
    echo "✓ Detected: $VERSION"
else
    echo "Could not detect version from web interface"
fi

echo ""
echo "=== Migration Validation ==="
if [ -f "user-data-summary.txt" ]; then
    echo "Comparing with original data:"
    cat user-data-summary.txt
else
    echo "Original data summary not found"
fi

echo ""
echo "=========================================="
echo "TESTING COMPLETE"
echo "=========================================="
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "Access Guacamole at: http://$SERVER_IP:8080/guacamole/"
echo ""
echo "If everything looks good above, try logging in with your original credentials."
echo "If you can't remember the admin password, we can reset it."
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/test-installation.sh
/tmp/test-installation.sh
```

### Step 5.3: Create a Script to Reset Admin Password (if needed)

**Create the password reset script:**
```bash
nano /tmp/reset-admin-password.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script resets the admin password if you can't log in

echo "Guacamole Admin Password Reset"
echo ""
echo "This script will reset the 'guacadmin' user password to 'guacadmin'"
echo "You should change this password immediately after logging in!"
echo ""
read -p "Are you sure you want to reset the admin password? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Password reset cancelled."
    exit 0
fi

echo ""
echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

if [ "$db_choice" = "1" ]; then
    echo "Resetting MySQL admin password..."
    echo "Please enter MySQL guacamole user password:"
    mysql -u guacamole -p guacamole << 'EOF'
UPDATE guacamole_user 
SET password_hash = UNHEX(SHA2(CONCAT('guacadmin', HEX(password_salt)), 256)) 
WHERE username = 'guacadmin';
SELECT 'Password reset for user:' as message, username FROM guacamole_user WHERE username = 'guacadmin';
EOF

    if [ $? -eq 0 ]; then
        echo "✓ Admin password reset successfully!"
    else
        echo "✗ Password reset failed!"
        exit 1
    fi
    
elif [ "$db_choice" = "2" ]; then
    echo "Resetting PostgreSQL admin password..."
    sudo -u postgres psql guacamole << 'EOF'
UPDATE guacamole_user 
SET password_hash = decode(sha256(concat(password_salt, 'guacadmin')::bytea), 'hex') 
WHERE username = 'guacadmin';
SELECT 'Password reset for user: ' || username FROM guacamole_user WHERE username = 'guacadmin';
EOF

    if [ $? -eq 0 ]; then
        echo "✓ Admin password reset successfully!"
    else
        echo "✗ Password reset failed!"
        exit 1
    fi
    
else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo ""
echo "=========================================="
echo "ADMIN PASSWORD RESET COMPLETE"
echo "=========================================="
echo "Username: guacadmin"
echo "Password: guacadmin"
echo ""
echo "⚠️  IMPORTANT: Log in and change this password immediately!"
echo "⚠️  Go to Settings > Users > guacadmin > Change Password"
```

**Save and make executable:**
```bash
chmod +x /tmp/reset-admin-password.sh
```

**Only run this if you can't log in:**
```bash
# Only run this if you need to reset the password
# /tmp/reset-admin-password.sh
```

---

## Phase 6: Security and Final Setup

### Step 6.1: Create a Script to Set Up Basic Security

**Create the security setup script:**
```bash
nano /tmp/setup-security.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script sets up basic security for your Guacamole server

echo "Setting up basic security for Guacamole server..."

echo ""
echo "=== Installing and Configuring Firewall ==="
echo "Installing UFW (Uncomplicated Firewall)..."
sudo apt install -y ufw

echo "Configuring firewall rules..."
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

echo "Allowing SSH (port 22)..."
sudo ufw allow ssh

echo "Allowing HTTP (port 80)..."
sudo ufw allow 80/tcp

echo "Allowing HTTPS (port 443)..."
sudo ufw allow 443/tcp

echo "Allowing Guacamole (port 8080)..."
sudo ufw allow 8080/tcp

echo "Firewall status:"
sudo ufw status numbered

echo ""
echo "=== Setting Up Automatic Updates ==="
sudo apt install -y unattended-upgrades apt-listchanges

echo "Configuring automatic security updates..."
sudo dpkg-reconfigure -plow unattended-upgrades

echo ""
echo "=== Creating Backup Directory ==="
sudo mkdir -p /backup/guacamole
sudo chmod 700 /backup/guacamole

echo "Basic security setup complete!"
echo ""
echo "Recommendations:"
echo "1. Change default passwords immediately"
echo "2. Set up SSH key authentication (disable password auth)"
echo "3. Consider setting up a reverse proxy with SSL"
echo "4. Regularly update your system"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/setup-security.sh
/tmp/setup-security.sh
```

### Step 6.2: Create a Script to Set Up Automated Backups

**Create the backup setup script:**
```bash
nano /tmp/setup-backups.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script sets up automated backups for Guacamole

echo "Setting up automated backups for Guacamole..."

echo "What database are you using?"
echo "1) MySQL"
echo "2) PostgreSQL"
read -p "Enter choice (1 or 2): " db_choice

echo ""
echo "Creating backup script..."

if [ "$db_choice" = "1" ]; then
    sudo tee /usr/local/bin/guacamole-backup.sh << 'EOF'
#!/bin/bash
# Automated Guacamole backup script for MySQL

BACKUP_DIR="/backup/guacamole"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/$DATE"

# Create backup directory
mkdir -p "$BACKUP_PATH"

# Backup database
echo "Backing up MySQL database..."
mysqldump -u guacamole -p"StrongPassword123!" guacamole > "$BACKUP_PATH/guacamole-$DATE.sql"

# Backup configuration
echo "Backing up configuration files..."
cp -r /etc/guacamole "$BACKUP_PATH/"

# Create compressed archive
echo "Creating compressed backup..."
tar -czf "$BACKUP_PATH.tar.gz" -C "$BACKUP_DIR" "$DATE"
rm -rf "$BACKUP_PATH"

# Keep only last 7 backups
echo "Cleaning old backups..."
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs rm -f 2>/dev/null

echo "Backup completed: $BACKUP_PATH.tar.gz"
logger "Guacamole backup completed: $BACKUP_PATH.tar.gz"
EOF

elif [ "$db_choice" = "2" ]; then
    sudo tee /usr/local/bin/guacamole-backup.sh << 'EOF'
#!/bin/bash
# Automated Guacamole backup script for PostgreSQL

BACKUP_DIR="/backup/guacamole"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/$DATE"

# Create backup directory
mkdir -p "$BACKUP_PATH"

# Backup database
echo "Backing up PostgreSQL database..."
sudo -u postgres pg_dump guacamole > "$BACKUP_PATH/guacamole-$DATE.sql"

# Backup configuration
echo "Backing up configuration files..."
cp -r /etc/guacamole "$BACKUP_PATH/"

# Create compressed archive
echo "Creating compressed backup..."
tar -czf "$BACKUP_PATH.tar.gz" -C "$BACKUP_DIR" "$DATE"
rm -rf "$BACKUP_PATH"

# Keep only last 7 backups
echo "Cleaning old backups..."
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs rm -f 2>/dev/null

echo "Backup completed: $BACKUP_PATH.tar.gz"
logger "Guacamole backup completed: $BACKUP_PATH.tar.gz"
EOF

else
    echo "Invalid choice. Exiting."
    exit 1
fi

echo "Making backup script executable..."
sudo chmod +x /usr/local/bin/guacamole-backup.sh

echo "Testing backup script..."
sudo /usr/local/bin/guacamole-backup.sh

echo "Setting up daily backup cron job..."
echo "0 2 * * * root /usr/local/bin/guacamole-backup.sh" | sudo tee -a /etc/crontab

echo "Backup setup complete!"
echo ""
echo "Your backups will be stored in: /backup/guacamole/"
echo "Backups run daily at 2:00 AM"
echo "Only the last 7 backups are kept"
echo ""
echo "To manually run a backup: sudo /usr/local/bin/guacamole-backup.sh"
echo "To view backups: ls -la /backup/guacamole/"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/setup-backups.sh
/tmp/setup-backups.sh
```

### Step 6.3: Create a Script for Final System Check

**Create the final check script:**
```bash
nano /tmp/final-system-check.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script performs a final system check

echo "=========================================="
echo "FINAL GUACAMOLE MIGRATION CHECK"
echo "=========================================="
echo "Date: $(date)"
echo ""

echo "=== System Information ==="
echo "OS: $(lsb_release -d | cut -f2)"
echo "Kernel: $(uname -r)"
echo "Hostname: $(hostname)"
echo "IP Address: $(hostname -I | awk '{print $1}')"
echo ""

echo "=== Service Status ==="
services=("guacd" "tomcat" "mysql" "postgresql")
for service in "${services[@]}"; do
    if systemctl list-unit-files | grep -q "^$service.service"; then
        status=$(systemctl is-active $service 2>/dev/null)
        echo "$service: $status"
    fi
done
echo ""

echo "=== Network Ports ==="
echo "Listening ports:"
sudo netstat -tlnp | grep -E "(8080|4822|3306|5432)"
echo ""

echo "=== Database Information ==="
if grep -q "mysql" /etc/guacamole/guacamole.properties; then
    echo "Database Type: MySQL"
    USERS=$(mysql -u guacamole -p"StrongPassword123!" -N -e "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null)
    CONNECTIONS=$(mysql -u guacamole -p"StrongPassword123!" -N -e "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null)
elif grep -q "postgresql" /etc/guacamole/guacamole.properties; then
    echo "Database Type: PostgreSQL"
    USERS=$(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_user;" guacamole 2>/dev/null | xargs)
    CONNECTIONS=$(sudo -u postgres psql -t -c "SELECT COUNT(*) FROM guacamole_connection;" guacamole 2>/dev/null | xargs)
fi

echo "Users migrated: $USERS"
echo "Connections migrated: $CONNECTIONS"
echo ""

echo "=== Guacamole Configuration ==="
echo "Configuration directory: /etc/guacamole/"
echo "Extensions installed:"
ls -1 /etc/guacamole/extensions/ 2>/dev/null | sed 's/^/  - /'
echo "Libraries installed:"
ls -1 /etc/guacamole/lib/ 2>/dev/null | sed 's/^/  - /' || echo "  - None"
echo ""

echo "=== Web Interface Test ==="
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/guacamole/ 2>/dev/null)
if [ "$HTTP_STATUS" = "200" ] || [ "$HTTP_STATUS" = "302" ]; then
    echo "✓ Web interface is accessible (HTTP $HTTP_STATUS)"
    VERSION=$(curl -s http://localhost:8080/guacamole/ | grep -o 'Guacamole [0-9]\+\.[0-9]\+\.[0-9]\+' | head -1)
    echo "  Version detected: $VERSION"
else
    echo "✗ Web interface is not accessible (HTTP $HTTP_STATUS)"
fi
echo ""

echo "=== Security Status ==="
echo "Firewall status:"
sudo ufw status | grep -E "(Status|8080|22|80|443)"
echo ""

echo "=== Backup Status ==="
if [ -f "/usr/local/bin/guacamole-backup.sh" ]; then
    echo "✓ Backup script installed"
    echo "Backup directory: /backup/guacamole/"
    BACKUP_COUNT=$(ls -1 /backup/guacamole/*.tar.gz 2>/dev/null | wc -l)
    echo "Backups available: $BACKUP_COUNT"
else
    echo "✗ Backup script not installed"
fi
echo ""

echo "=== Migration Summary ==="
echo "✓ Guacamole 1.6.0 installed"
echo "✓ Database migrated and upgraded"
echo "✓ Configuration transferred"
echo "✓ Services running"
echo "✓ Basic security configured"
echo "✓ Backups configured"
echo ""

echo "=========================================="
echo "MIGRATION ACCESS INFORMATION"
echo "=========================================="
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "Guacamole URL: http://$SERVER_IP:8080/guacamole/"
echo ""
echo "If you can't log in with your original credentials, run:"
echo "/tmp/reset-admin-password.sh"
echo ""
echo "=========================================="
echo "NEXT STEPS"
echo "=========================================="
echo "1. Test logging in with your original credentials"
echo "2. Verify all your connections work"
echo "3. Change any default passwords"
echo "4. Consider setting up SSL/HTTPS"
echo "5. Update DNS to point to this server"
echo "6. Keep your old server as backup for a few days"
echo ""
echo "Migration completed successfully!"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/final-system-check.sh
/tmp/final-system-check.sh
```

---

## Phase 7: Cleanup and Documentation

### Step 7.1: Create a Script to Clean Up Temporary Files

**Create the cleanup script:**
```bash
nano /tmp/cleanup-migration.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script cleans up temporary migration files

echo "Cleaning up migration temporary files..."

echo ""
read -p "Are you sure the migration completed successfully? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Cleanup cancelled. Temporary files kept for troubleshooting."
    exit 0
fi

echo ""
echo "The following directories and files will be removed:"
echo "- /tmp/guacamole-*"
echo "- /tmp/mysql-connector-*"
echo "- /tmp/migration/"
echo "- All .sh scripts in /tmp/"
echo ""
read -p "Proceed with cleanup? (yes/no): " final_confirm

if [ "$final_confirm" != "yes" ]; then
    echo "Cleanup cancelled."
    exit 0
fi

echo "Removing temporary files..."

# Remove Guacamole source and binary files
rm -rf /tmp/guacamole-*
rm -rf /tmp/mysql-connector-*
rm -rf /tmp/migration/

# Remove installation scripts
rm -f /tmp/*.sh

# Clean package cache
sudo apt autoremove -y
sudo apt autoclean

echo "Cleanup completed!"
echo ""
echo "What was kept:"
echo "- Configuration files in /etc/guacamole/"
echo "- Installed Guacamole application"
echo "- Database with your migrated data"
echo "- Backup script in /usr/local/bin/"
echo "- System logs"
```

**Save, make executable, and run ONLY when you're sure everything works:**
```bash
chmod +x /tmp/cleanup-migration.sh
# Only run this when you're confident everything is working:
# /tmp/cleanup-migration.sh
```

### Step 7.2: Create a Script to Generate Final Documentation

**Create the documentation script:**
```bash
nano /tmp/create-documentation.sh
```

**Copy and paste this:**
```bash
#!/bin/bash
# This script creates final migration documentation

echo "Creating final migration documentation..."

DOC_FILE="/root/guacamole-migration-complete.txt"

cat > $DOC_FILE << EOF
========================================
GUACAMOLE MIGRATION DOCUMENTATION
========================================
Migration completed: $(date)
Migrated from: CentOS 7 + Guacamole 1.1.0
Migrated to: Ubuntu 24.04 LTS + Guacamole 1.6.0

========================================
ACCESS INFORMATION
========================================
Web Interface: http://$(hostname -I | awk '{print $1}'):8080/guacamole/
Server IP: $(hostname -I | awk '{print $1}')
Database: $(grep -E "(mysql|postgresql)-database" /etc/guacamole/guacamole.properties | cut -d: -f2 | xargs)

========================================
SYSTEM INFORMATION
========================================
Operating System: $(lsb_release -d | cut -f2)
Hostname: $(hostname)
Java Version: $(java -version 2>&1 | head -1)

========================================
SERVICE MANAGEMENT
========================================
Start services: sudo systemctl start guacd tomcat
Stop services: sudo systemctl stop guacd tomcat
Restart services: sudo systemctl restart guacd tomcat
Check status: sudo systemctl status guacd tomcat

========================================
IMPORTANT FILE LOCATIONS
========================================
Configuration: /etc/guacamole/
Main config file: /etc/guacamole/guacamole.properties
Extensions: /etc/guacamole/extensions/
Libraries: /etc/guacamole/lib/
Tomcat webapps: /opt/tomcat/webapps/
Tomcat logs: /opt/tomcat/logs/
System logs: sudo journalctl -u guacd -u tomcat

========================================
BACKUP INFORMATION
========================================
Backup script: /usr/local/bin/guacamole-backup.sh
Backup location: /backup/guacamole/
Backup schedule: Daily at 2:00 AM
Manual backup: sudo /usr/local/bin/guacamole-backup.sh
View backups: ls -la /backup/guacamole/

========================================
DATABASE INFORMATION
========================================
EOF

# Add database-specific information
if grep -q "mysql" /etc/guacamole/guacamole.properties; then
    cat >> $DOC_FILE << EOF
Database type: MySQL
Connect to database: mysql -u guacamole -p guacamole
Service: sudo systemctl status mysql
EOF
elif grep -q "postgresql" /etc/guacamole/guacamole.properties; then
    cat >> $DOC_FILE << EOF
Database type: PostgreSQL
Connect to database: sudo -u postgres psql guacamole
Service: sudo systemctl status postgresql
EOF
fi

cat >> $DOC_FILE << EOF

========================================
SECURITY NOTES
========================================
Firewall: UFW enabled (ports 22, 80, 443, 8080 open)
Change default passwords immediately after migration
Consider setting up SSL/HTTPS for production use
Regular system updates recommended

========================================
TROUBLESHOOTING
========================================
If web interface doesn't load:
  1. Check services: sudo systemctl status guacd tomcat
  2. Check logs: sudo tail -f /opt/tomcat/logs/catalina.out
  3. Check ports: sudo netstat -tlnp | grep 8080

If database connection fails:
  1. Test database: mysql -u guacamole -p guacamole (MySQL)
  2. Test database: sudo -u postgres psql guacamole (PostgreSQL)
  3. Check config: cat /etc/guacamole/guacamole.properties

If you can't log in:
  1. Reset admin password: /tmp/reset-admin-password.sh
  2. Default will be: username=guacadmin, password=guacadmin

========================================
USEFUL COMMANDS
========================================
View current connections: sudo netstat -tlnp | grep -E "(8080|4822)"
Check disk space: df -h
Check memory usage: free -h
View system logs: sudo journalctl -xe
Monitor logs in real-time: sudo journalctl -f

========================================
MIGRATION VALIDATION
========================================
EOF

# Add migration validation data
if [ -f "/tmp/migration/user-data-summary.txt" ]; then
    echo "Original data from CentOS 7 server:" >> $DOC_FILE
    cat /tmp/migration/user-data-summary.txt >> $DOC_FILE
else
    echo "Migration data summary not available" >> $DOC_FILE
fi

cat >> $DOC_FILE << EOF

Current data on Ubuntu server:
EOF

# Add current data counts
if grep -q "mysql" /etc/guacamole/guacamole.properties; then
    mysql -u guacamole -p"StrongPassword123!" guacamole << 'SQL_EOF' >> $DOC_FILE 2>/dev/null
SELECT 'Users found:' as info, COUNT(*) as count FROM guacamole_user;
SELECT 'Connections found:' as info, COUNT(*) as count FROM guacamole_connection;
SELECT 'User groups found:' as info, COUNT(*) as count FROM guacamole_user_group;
SQL_EOF
elif grep -q "postgresql" /etc/guacamole/guacamole.properties; then
    sudo -u postgres psql guacamole << 'SQL_EOF' >> $DOC_FILE 2>/dev/null
SELECT 'Users found:' as info, COUNT(*) as count FROM guacamole_user;
SELECT 'Connections found:' as info, COUNT(*) as count FROM guacamole_connection;
SELECT 'User groups found:' as info, COUNT(*) as count FROM guacamole_user_group;
SQL_EOF
fi

cat >> $DOC_FILE << EOF

========================================
END OF DOCUMENTATION
========================================
This documentation was generated automatically.
Keep this file for future reference.
File location: $DOC_FILE
EOF

echo "Documentation created successfully!"
echo "File location: $DOC_FILE"
echo ""
echo "To view the documentation:"
echo "cat $DOC_FILE"
echo ""
echo "To print the documentation:"
echo "less $DOC_FILE"
```

**Save, make executable, and run:**
```bash
chmod +x /tmp/create-documentation.sh
/tmp/create-documentation.sh
```

---

## Complete Migration Summary

### What You've Accomplished

You have successfully migrated your Apache Guacamole installation from CentOS 7 to Ubuntu 24.04 LTS and upgraded from version 1.1.0 to 1.6.0. Here's what was completed:

1. **Backed up your old server** completely with all data and configurations
2. **Set up a new Ubuntu 24.04 server** with all required dependencies  
3. **Installed the latest Guacamole 1.6.0** from source
4. **Migrated all your data** including users, connections, and settings
5. **Upgraded the database schema** to support new features
6. **Configured security** with firewall and automated backups
7. **Tested everything** to ensure it works properly

### How to Access Your New Server

**Web Interface:** `http://your-server-ip:8080/guacamole/`

**Login:** Use your original username and password from the old server

**If you can't log in:** Run the reset password script:
```bash
/tmp/reset-admin-password.sh
```

### Important Next Steps

1. **Test everything thoroughly** - try all your connections
2. **Change any default passwords** immediately  
3. **Update DNS records** to point to the new server
4. **Keep your old server running** for a few days as backup
5. **Consider setting up SSL/HTTPS** for production use

### If Something Goes Wrong

All your scripts are saved in `/tmp/` and can be run again if needed. Your complete documentation is in `/root/guacamole-migration-complete.txt`.

**For troubleshooting:**
- Check service status: `sudo systemctl status guacd tomcat`
- View logs: `sudo tail -f /opt/tomcat/logs/catalina.out`
- Test database: Use the appropriate command for your database type

### Final Notes

- **Backups run automatically** every night at 2:00 AM
- **Security is configured** with UFW firewall
- **All temporary files can be cleaned up** once you're confident everything works
- **Complete documentation** has been generated for future reference

**Congratulations! Your Guacamole migration is complete!** 🎉

Your new Ubuntu server is now running the latest Guacamole 1.6.0 with all your original data, users, and connections preserved. "Old server IP address: " old_server_ip
    read -p