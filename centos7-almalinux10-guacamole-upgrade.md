# Complete Upgrade Guide: CentOS 7 to AlmaLinux with Apache Guacamole 1.1.0 → 1.6.0

## Overview

This guide covers upgrading a CentOS 7 server running Apache Guacamole 1.1.0 to AlmaLinux 9 with Apache Guacamole 1.6.0. This is a complex multi-step process that requires careful planning and backup procedures.

**Current State:** CentOS 7 + Apache Guacamole 1.1.0  
**Target State:** AlmaLinux 10.0 + Apache Guacamole 1.6.0

## Important Notes

- **CentOS 7 reached end-of-life on June 30, 2024** - no more security updates
- This is a **three-stage OS upgrade**: CentOS 7 → AlmaLinux 8 → AlmaLinux 9 → AlmaLinux 10.0
- **Apache Guacamole 1.6.0** (latest) was released on June 22, 2025
- **AlmaLinux 10.0** "Purple Lion" was released on May 27, 2025
- **Estimated downtime:** 3-5 hours depending on system size and complexity
- **Risk level:** High - complete system and application backup mandatory

## Phase 5: OS Upgrade - AlmaLinux 9 to AlmaLinux 10.0

### 5.1 Prepare AlmaLinux 9 for Final Upgrade

```bash
# Update AlmaLinux 9 to latest
dnf update -y

# Install ELevate for 9→10 upgrade
dnf install -y http://repo.almalinux.org/elevate/elevate-release-latest-el9.noarch.rpm
dnf install -y leapp-upgrade leapp-data-almalinux
```

### 5.2 Pre-Upgrade Check for AlmaLinux 10

```bash
# Run pre-upgrade analysis
leapp preupgrade

# Review report and address any issues
cat /var/log/leapp/leapp-report.txt

# Handle common issues for progressive upgrades from CentOS 7
# Remove any leftover packages from previous upgrades
dnf remove $(rpm -qa | grep -E "(elevate|leapp)" | grep -v $(rpm -q leapp-upgrade leapp-data-almalinux))

# Clean up configuration files
sed -i '/exclude.*elevate\|exclude.*leapp/d' /etc/yum.conf
sed -i '/exclude.*elevate\|exclude.*leapp/d' /etc/dnf/dnf.conf
```

### 5.3 Upgrade to AlmaLinux 10.0

```bash
# Start upgrade to AlmaLinux 10.0
leapp upgrade

# System will reboot automatically
# Monitor console access during upgrade
```

### 5.4 Verify AlmaLinux 10.0 Installation

```bash
# Verify final OS version
cat /etc/redhat-release
# Should show: AlmaLinux release 10.0 (Purple Lion)

cat /etc/os-release

# Clean up upgrade artifacts
dnf remove -y elevate-release leapp-upgrade leapp-data-almalinux

# Update to latest packages
dnf update -y

# Check for any remaining old packages
rpm -qa | grep -E "el[789]" | head -20
```

---

## Pre-Upgrade Planning & Requirements

### System Requirements
- **Minimum 20 GB free disk space** (more recommended based on system size)
- **Root or sudo access**
- **Stable internet connection**
- **Console access** (IPMI, KVM, or physical access)
- **Redundant power supply** if available

### Pre-Upgrade Checklist
- [ ] Create complete system backup
- [ ] Document current Guacamole configuration
- [ ] Test backup restoration procedure
- [ ] Schedule maintenance window
- [ ] Notify users of downtime
- [ ] Prepare rollback plan

---

## Phase 1: System Backup and Documentation

### 1.1 Create Complete System Backup

```bash
# Create backup directory
mkdir -p /backup/pre-upgrade-$(date +%Y%m%d)
cd /backup/pre-upgrade-$(date +%Y%m%d)

# System configuration backup
tar -czf system-config.tar.gz /etc /home /var/log

# Database backup (if using local database)
# For MySQL/MariaDB:
mysqldump -u root -p --all-databases > guacamole-db-backup.sql

# For PostgreSQL:
pg_dumpall > guacamole-db-backup.sql

# Guacamole specific backups
cp -r /etc/guacamole ./guacamole-config-backup
cp -r /usr/share/tomcat/webapps/guacamole ./guacamole-webapp-backup
systemctl status guacd > guacd-service-status.txt
systemctl status tomcat > tomcat-service-status.txt
```

### 1.2 Document Current Configuration

```bash
# System information
cat /etc/redhat-release > system-info.txt
uname -a >> system-info.txt
df -h >> system-info.txt
free -h >> system-info.txt

# Network configuration
ip addr show > network-config.txt
cat /etc/resolv.conf >> network-config.txt

# Services and ports
ss -tulpn > ports-services.txt
systemctl list-enabled > enabled-services.txt

# Guacamole version and configuration
cat /etc/guacamole/guacamole.properties > current-guac-config.txt
ls -la /etc/guacamole/ > guac-files.txt

# Package list
rpm -qa | sort > installed-packages.txt
```

---

## Phase 2: Pre-Upgrade System Preparation

### 2.1 Update Current System

```bash
# Update system to latest CentOS 7 packages
yum update -y

# Clean up system
yum autoremove -y
yum clean all

# Check for any broken packages
rpm -Va | grep "^..5" || echo "No configuration file changes detected"
```

### 2.2 Stop Guacamole Services

```bash
# Stop Guacamole services
systemctl stop tomcat
systemctl stop guacd

# Verify services are stopped
systemctl status tomcat
systemctl status guacd
```

### 2.3 Prepare for OS Upgrade

```bash
# Check available disk space
df -h

# Remove unnecessary packages that might cause conflicts
yum remove -y $(package-cleanup --leaves)

# Check for custom repositories that might conflict
ls /etc/yum.repos.d/
```

---

## Phase 3: OS Upgrade - CentOS 7 to AlmaLinux 8

### 3.1 Install ELevate Migration Tool

```bash
# Install EPEL repository if not already installed
yum install -y epel-release

# Install ELevate repository
yum install -y http://repo.almalinux.org/elevate/elevate-release-latest-el7.noarch.rpm

# Install Leapp migration tools
yum install -y leapp-upgrade leapp-data-almalinux
```

### 3.2 Run Pre-Upgrade Analysis

```bash
# Run pre-upgrade check
leapp preupgrade

# Review the pre-upgrade report
cat /var/log/leapp/leapp-report.txt
```

### 3.3 Address Common Issues

```bash
# Common fixes for CentOS 7 upgrades
# Remove problematic kernel module
sudo rmmod pata_acpi 2>/dev/null || true

# Enable root login for upgrade process
echo "PermitRootLogin yes" | sudo tee -a /etc/ssh/sshd_config

# Answer common upgrade questions
leapp answer --section remove_pam_pkcs11_module_check.confirm=True
```

### 3.4 Perform OS Upgrade to AlmaLinux 8

```bash
# Start the upgrade process
leapp upgrade

# System will reboot twice automatically
# Monitor console during upgrade process
```

### 3.5 Post-Upgrade Verification (AlmaLinux 8)

```bash
# Verify upgrade success
cat /etc/redhat-release
cat /etc/os-release

# Check for upgrade-related packages that can be removed
rpm -qa | grep -E "(elevate|leapp)"

# Clean up upgrade packages
dnf remove -y elevate-release leapp-upgrade leapp-data-almalinux
```

---

## Phase 4: OS Upgrade - AlmaLinux 8 to AlmaLinux 9

### 4.1 Prepare AlmaLinux 8 for Next Upgrade

```bash
# Update AlmaLinux 8 to latest
dnf update -y

# Install ELevate for 8→9 upgrade
dnf install -y http://repo.almalinux.org/elevate/elevate-release-latest-el8.noarch.rpm
dnf install -y leapp-upgrade leapp-data-almalinux
```

### 4.2 Pre-Upgrade Check for AlmaLinux 9

```bash
# Run pre-upgrade analysis
leapp preupgrade

# Review report and address any issues
cat /var/log/leapp/leapp-report.txt
```

### 4.3 Upgrade to AlmaLinux 9

```bash
# Start upgrade to AlmaLinux 9
leapp upgrade

# System will reboot automatically
# Monitor console access during upgrade
```

### 4.4 Verify AlmaLinux 9 Installation

```bash
# Verify final OS version
cat /etc/redhat-release
# Should show: AlmaLinux release 9.x

# Clean up upgrade artifacts
dnf remove -y elevate-release leapp-upgrade leapp-data-almalinux

# Update to latest packages
dnf update -y
```

---

## Phase 6: Guacamole Upgrade Preparation

### 5.1 Install Required Dependencies for Guacamole 1.6.0

```bash
# Install development tools
dnf groupinstall -y "Development Tools"

# Install Guacamole build dependencies
dnf install -y cairo-devel libjpeg-turbo-devel libpng-devel \
               libtool uuid-devel freerdp-devel pango-devel \
               libssh2-devel libtelnet-devel libvncserver-devel \
               pulseaudio-libs-devel openssl-devel libvorbis-devel \
               libwebp-devel libwebsockets-devel

# Install Java 11 (required for Guacamole 1.6.0)
dnf install -y java-11-openjdk java-11-openjdk-devel

# Install Tomcat 9
dnf install -y tomcat

# Install database connector (choose based on your database)
# For MySQL/MariaDB:
dnf install -y mysql-connector-java

# For PostgreSQL:
dnf install -y postgresql-jdbc
```

### 5.2 Download Guacamole 1.6.0 Source

```bash
# Create build directory
mkdir -p /tmp/guacamole-upgrade
cd /tmp/guacamole-upgrade

# Download Guacamole 1.6.0 server
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-server-1.6.0.tar.gz
wget https://downloads.apache.org/guacamole/1.6.0/source/guacamole-server-1.6.0.tar.gz.asc

# Download Guacamole 1.6.0 client
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-1.6.0.war
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-1.6.0.war.asc

# Verify signatures (optional but recommended)
wget https://downloads.apache.org/guacamole/KEYS
gpg --import KEYS
gpg --verify guacamole-server-1.6.0.tar.gz.asc
gpg --verify guacamole-1.6.0.war.asc
```

---

## Phase 7: Guacamole Server Upgrade

### 6.1 Remove Old Guacamole Server

```bash
# Stop services
systemctl stop guacd

# Remove old guacd binary (backup first)
cp /usr/local/sbin/guacd /backup/pre-upgrade-$(date +%Y%m%d)/guacd-old
rm -f /usr/local/sbin/guacd
```

### 6.2 Build and Install Guacamole Server 1.6.0

```bash
# Extract and build new server
cd /tmp/guacamole-upgrade
tar -xzf guacamole-server-1.6.0.tar.gz
cd guacamole-server-1.6.0

# Configure build
./configure --with-init-dir=/etc/init.d

# Compile and install
make
make install

# Update library cache
ldconfig
```

### 6.3 Update Guacamole Server Service

```bash
# Create systemd service file for AlmaLinux 9
cat > /etc/systemd/system/guacd.service << 'EOF'
[Unit]
Description=Guacamole Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/sbin/guacd
User=daemon
Group=daemon

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and enable service
systemctl daemon-reload
systemctl enable guacd
```

---

## Phase 8: Guacamole Web Application Upgrade

### 7.1 Backup and Replace Web Application

```bash
# Stop Tomcat
systemctl stop tomcat

# Backup current web application
cp /usr/share/tomcat/webapps/guacamole.war /backup/pre-upgrade-$(date +%Y%m%d)/

# Remove old application
rm -rf /usr/share/tomcat/webapps/guacamole*

# Install new web application
cp /tmp/guacamole-upgrade/guacamole-1.6.0.war /usr/share/tomcat/webapps/guacamole.war

# Set correct ownership
chown tomcat:tomcat /usr/share/tomcat/webapps/guacamole.war
```

### 7.2 Update Database Schema (If Required)

**Important:** Guacamole 1.6.0 includes database schema changes for new audit permissions.

```bash
# Download database upgrade scripts
cd /tmp/guacamole-upgrade
wget https://raw.githubusercontent.com/apache/guacamole-client/main/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/upgrade/upgrade-pre-1.6.0.sql

# For MySQL/MariaDB:
mysql -u root -p guacamole_db < upgrade-pre-1.6.0.sql

# For PostgreSQL:
psql -U guacamole_user -d guacamole_db -f upgrade-pre-1.6.0.sql
```

### 7.3 Update Configuration Files

```bash
# Review and update guacamole.properties if needed
vi /etc/guacamole/guacamole.properties

# Ensure database connector is properly linked
# For MySQL:
ln -sf /usr/share/java/mysql-connector-java.jar /etc/guacamole/lib/

# For PostgreSQL:
ln -sf /usr/share/java/postgresql-jdbc.jar /etc/guacamole/lib/
```

---

## Phase 9: Extensions Update

### 8.1 Download Updated Extensions

```bash
# Download compatible extensions for 1.6.0
cd /tmp/guacamole-upgrade

# LDAP Extension (if used)
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-ldap-1.6.0.tar.gz

# TOTP Extension (if used)
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-totp-1.6.0.tar.gz

# Duo Extension (if used) - NOTE: Requires migration to Duo v4
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-duo-1.6.0.tar.gz

# Quick Connect Extension (if used)
wget https://downloads.apache.org/guacamole/1.6.0/binary/guacamole-auth-quickconnect-1.6.0.tar.gz
```

### 8.2 Install Updated Extensions

```bash
# Backup current extensions
cp /etc/guacamole/extensions/* /backup/pre-upgrade-$(date +%Y%m%d)/extensions/

# Remove old extensions
rm -f /etc/guacamole/extensions/*.jar

# Extract and install new extensions
cd /tmp/guacamole-upgrade

# Install each extension you use:
tar -xzf guacamole-auth-ldap-1.6.0.tar.gz
cp guacamole-auth-ldap-1.6.0/guacamole-auth-ldap-1.6.0.jar /etc/guacamole/extensions/

# Repeat for other extensions as needed
```

### 8.3 Update Duo Configuration (If Used)

**Important:** Duo v4 requires different configuration parameters.

```bash
# Review and update Duo configuration in guacamole.properties
# Old v2 parameters need to be replaced with v4 equivalents
# Refer to Guacamole 1.6.0 documentation for new parameter names
vi /etc/guacamole/guacamole.properties
```

---

## Phase 10: Service Configuration and Testing

### 9.1 Configure Firewall (firewalld replaces iptables)

```bash
# Configure firewall for AlmaLinux 9
systemctl enable firewalld
systemctl start firewalld

# Open required ports
firewall-cmd --permanent --add-port=8080/tcp  # Tomcat
firewall-cmd --permanent --add-port=4822/tcp  # Guacd
firewall-cmd --reload

# Verify firewall rules
firewall-cmd --list-all
```

### 9.2 Set Correct Permissions

```bash
# Set ownership and permissions
chown -R tomcat:tomcat /etc/guacamole
chmod -R 600 /etc/guacamole/guacamole.properties
chmod 755 /etc/guacamole
chmod 755 /etc/guacamole/extensions
chmod 755 /etc/guacamole/lib
```

---

## Phase 11: Service Startup and Verification

### 10.1 Start Services

```bash
# Start services in order
systemctl start guacd
systemctl status guacd

# Wait a moment, then start Tomcat
sleep 5
systemctl start tomcat
systemctl status tomcat

# Enable services for automatic startup
systemctl enable guacd
systemctl enable tomcat
```

### 10.2 Verify Installation

```bash
# Check service status
systemctl status guacd
systemctl status tomcat

# Check listening ports
ss -tulpn | grep -E "(4822|8080)"

# Check logs for errors
tail -f /var/log/tomcat/catalina.out
tail -f /var/log/messages | grep guacd

# Test web interface access
curl -I http://localhost:8080/guacamole/
```

### 10.3 Functional Testing

1. **Access Guacamole Web Interface**
   - Navigate to `http://your-server:8080/guacamole/`
   - Verify login page appears correctly

2. **Test User Authentication**
   - Login with existing user account
   - Verify authentication against your configured method (database/LDAP/etc.)

3. **Test Connection Functionality**
   - Create a test connection
   - Verify connection to remote desktop/server works
   - Test clipboard, file transfer, and other features

4. **Test Extensions**
   - Verify TOTP/MFA if configured
   - Test LDAP authentication if used
   - Verify any custom extensions

---

## Phase 12: Post-Upgrade Cleanup and Optimization

### 11.1 System Cleanup

```bash
# Remove temporary build files
rm -rf /tmp/guacamole-upgrade

# Clean package cache
dnf clean all

# Remove old kernel versions (keep 2-3 recent ones)
dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)

# Update system to latest packages
dnf update -y
```

### 11.2 Security Hardening

```bash
# Update SELinux contexts if needed
restorecon -R /etc/guacamole
restorecon -R /usr/share/tomcat

# Review and tighten firewall rules
firewall-cmd --list-all

# Check for any security updates
dnf check-update --security
```

### 11.3 Performance Optimization

```bash
# Optimize Tomcat memory settings
vi /etc/tomcat/tomcat.conf
# Add: JAVA_OPTS="-Xms1024m -Xmx2048m -XX:+UseG1GC"

# Restart Tomcat to apply changes
systemctl restart tomcat
```

---

## Phase 13: Backup and Documentation

### 12.1 Create Post-Upgrade Backup

```bash
# Create backup of new configuration
mkdir -p /backup/post-upgrade-$(date +%Y%m%d)
cd /backup/post-upgrade-$(date +%Y%m%d)

# Backup new configurations
tar -czf new-system-config.tar.gz /etc /home

# Database backup
mysqldump -u root -p --all-databases > guacamole-db-new.sql

# Guacamole configuration
cp -r /etc/guacamole ./guacamole-config-new
```

### 13.2 Update Documentation

```bash
# Document new system information
cat /etc/redhat-release > new-system-info.txt
echo "AlmaLinux Version: 10.0 (Purple Lion)" >> new-system-info.txt
echo "Guacamole Version: 1.6.0" >> new-system-info.txt
echo "Upgrade completed: $(date)" >> new-system-info.txt

# Document configuration changes
diff /backup/pre-upgrade-*/guacamole-config-backup/guacamole.properties \
     /etc/guacamole/guacamole.properties > config-changes.txt
```

---

## Troubleshooting Guide

### Common Issues and Solutions

**Issue: Services fail to start after upgrade**
```bash
# Check service logs
journalctl -u guacd -f
journalctl -u tomcat -f

# Check for permission issues
ls -la /etc/guacamole/
restorecon -R /etc/guacamole
```

**Issue: Database connection fails**
```bash
# Verify database connectivity
mysql -u guacamole_user -p guacamole_db
# Or for PostgreSQL:
psql -U guacamole_user -d guacamole_db

# Check database connector
ls -la /etc/guacamole/lib/
```

**Issue: Web application not loading**
```bash
# Check Tomcat logs
tail -f /var/log/tomcat/catalina.out

# Verify war file deployment
ls -la /usr/share/tomcat/webapps/
```

**Issue: Extensions not working**
```bash
# Check extension files
ls -la /etc/guacamole/extensions/

# Review configuration
cat /etc/guacamole/guacamole.properties | grep -v "^#"
```

---

## Rollback Procedure

If issues occur that cannot be resolved:

### 1. Service Rollback
```bash
# Stop services
systemctl stop tomcat guacd

# Restore old Guacamole files
cp /backup/pre-upgrade-*/guacd-old /usr/local/sbin/guacd
cp /backup/pre-upgrade-*/guacamole.war /usr/share/tomcat/webapps/
cp -r /backup/pre-upgrade-*/guacamole-config-backup/* /etc/guacamole/

# Restore database
mysql -u root -p < /backup/pre-upgrade-*/guacamole-db-backup.sql

# Start services
systemctl start guacd tomcat
```

### 2. Full System Rollback
If OS upgrade caused issues, restore from complete system backup or rebuild from known-good state.

---

## New Features in Guacamole 1.6.0

- **Improved rendering performance** for RDP connections
- **Enhanced Docker support**
- **Configurable case sensitivity** for user authentication
- **Batch connection import** functionality
- **Duo v4 support** (replaces v2)
- **Time limits** for user logins and connection usage
- **Enhanced clipboard security** (contents hidden by default)

---

## Maintenance Recommendations

### Regular Maintenance Tasks
- **Weekly:** Check system and application logs
- **Monthly:** Apply security updates with `dnf update --security`
- **Quarterly:** Full system update and backup verification
- **Annually:** Review security configuration and access controls

### Monitoring Setup
```bash
# Set up log monitoring
tail -f /var/log/tomcat/catalina.out | grep ERROR &
tail -f /var/log/messages | grep guacd &

# Check service status regularly
systemctl status guacd tomcat
```

---

## Summary

This comprehensive upgrade process migrates your system from end-of-life CentOS 7 to modern AlmaLinux 10.0 while updating Apache Guacamole from version 1.1.0 to the latest 1.6.0. The process ensures:

- **Security:** Moving to the latest supported OS with regular security updates through 2035
- **Performance:** Leveraging improved rendering and optimization in Guacamole 1.6.0  
- **Compatibility:** Maintaining existing configurations and connections
- **Reliability:** Following best practices for backup and rollback procedures

**Expected Benefits:**
- Security updates through May 31, 2035 (AlmaLinux 10.0)
- Active support through May 31, 2030
- Improved performance and new features in both OS and Guacamole
- Better hardware support with x86-64-v2 architecture support
- Enhanced authentication and session management
- Post-quantum cryptography support

**Post-Upgrade:** Monitor the system closely for the first week and maintain regular backups of both system and Guacamole configurations.

## AlmaLinux 10.0 New Features

- **Extended hardware support:** x86-64-v2 architecture support for older hardware
- **Enhanced security:** Post-quantum cryptography support and updates to SELinux and OpenSSH
- **Improved virtualization:** SPICE support enabled and KVM for IBM POWER
- **Frame pointers enabled** for system-wide real-time tracing and profiling
- **Updated development tools:** New versions of programming languages, toolchains, and compilers
- **Long-term support:** Active support until May 31, 2030, security support until May 31, 2035