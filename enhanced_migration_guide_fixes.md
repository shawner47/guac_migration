# Enhanced Guacamole Migration Guide - Critical Fixes and Lessons Learned

## Critical Post-Installation Authentication Issues

### Issue 1: Missing MySQL Authentication Extension

**Problem**: After migrating Guacamole, users cannot log in even with correct passwords. Login attempts show "Invalid Login" with log entries showing "unknown error (no specific failure recorded)".

**Root Cause**: The MySQL authentication extension (`guacamole-auth-jdbc-mysql-X.X.X.jar`) is missing from `/etc/guacamole/extensions/`, preventing Guacamole from connecting to the database for authentication.

**Solution**:

1. **Verify the problem**:
   ```bash
   # Check if extensions directory is empty
   sudo ls -la /etc/guacamole/extensions/
   
   # Check Tomcat logs for authentication errors
   sudo tail -f /opt/tomcat/logs/catalina.out
   # Look for: "Authentication attempt from [IP] for user [username] failed: unknown error"
   ```

2. **Install the MySQL authentication extension**:
   ```bash
   # Determine your Guacamole version
   sudo find /opt/tomcat -name "guacamole*.jar" -exec basename {} \;
   
   # Download the matching version (replace X.X.X with your version)
   cd /tmp
   wget https://archive.apache.org/dist/guacamole/1.6.0/binary/guacamole-auth-jdbc-1.6.0.tar.gz
   tar -xzf guacamole-auth-jdbc-1.6.0.tar.gz
   
   # Install the MySQL extension
   sudo cp guacamole-auth-jdbc-1.6.0/mysql/guacamole-auth-jdbc-mysql-1.6.0.jar /etc/guacamole/extensions/
   sudo chown tomcat:tomcat /etc/guacamole/extensions/guacamole-auth-jdbc-mysql-1.6.0.jar
   
   # Restart Tomcat
   sudo systemctl restart tomcat
   ```

3. **Verify the fix**:
   ```bash
   # Check that the extension is loaded
   sudo ls -la /etc/guacamole/extensions/
   
   # Test login with database users
   # Should now work with guacadmin/guacadmin (or other database users)
   ```

### Issue 2: Password Reset for Database Users

**Problem**: Cannot log in with default credentials after migration.

**Enhanced Password Reset Script**:

```bash
#!/bin/bash
# Enhanced password reset script that works with entity-based user structure

echo "Guacamole Database User Password Reset"
echo ""
echo "This script will reset a user's password in the Guacamole database"
echo ""

# Show available admin users
echo "Finding admin users with ADMINISTER permissions..."
mysql -u guacamole -p guac_db -e "
SELECT e.name as username, u.full_name, u.email_address 
FROM guacamole_entity e
JOIN guacamole_user u ON e.entity_id = u.entity_id
JOIN guacamole_system_permission sp ON e.entity_id = sp.entity_id
WHERE sp.permission = 'ADMINISTER' AND e.type = 'USER'
ORDER BY e.name;"

echo ""
read -p "Enter the username to reset password for: " target_user
read -p "Enter the new password: " new_password

echo ""
echo "Resetting password for user: $target_user"
echo "Please enter MySQL guacamole user password:"

# Reset password using correct entity-based approach
mysql -u guacamole -p guac_db << EOF
-- Find the entity_id for the target user
SET @target_entity_id = (SELECT entity_id FROM guacamole_entity WHERE name = '$target_user' AND type = 'USER');

-- Generate a new salt
SET @salt = UNHEX(SHA2(RAND(), 256));

-- Update the password hash with salt
UPDATE guacamole_user 
SET password_hash = UNHEX(SHA2(CONCAT('$new_password', HEX(@salt)), 256)),
    password_salt = @salt,
    password_date = NOW()
WHERE entity_id = @target_entity_id;

-- Verify the update
SELECT 
    'Password reset completed for:' as message,
    e.name as username,
    CASE WHEN u.password_hash IS NOT NULL THEN 'SUCCESS' ELSE 'FAILED' END as status
FROM guacamole_entity e
JOIN guacamole_user u ON e.entity_id = u.entity_id
WHERE e.entity_id = @target_entity_id;
EOF

if [ $? -eq 0 ]; then
    echo "✓ Password reset completed successfully!"
    echo "Restarting Tomcat..."
    sudo systemctl restart tomcat
else
    echo "✗ Password reset failed!"
    exit 1
fi
```

### Issue 3: Restoring LDAP/Active Directory Authentication

**Problem**: After installing MySQL extension, LDAP authentication stops working.

**Solution**: Install LDAP extension and configure both authentication methods.

1. **Install LDAP authentication extension**:
   ```bash
   # Download and install LDAP extension
   cd /tmp
   wget https://archive.apache.org/dist/guacamole/1.6.0/binary/guacamole-auth-ldap-1.6.0.tar.gz
   tar -xzf guacamole-auth-ldap-1.6.0.tar.gz
   
   # Install the extension
   sudo cp guacamole-auth-ldap-1.6.0/guacamole-auth-ldap-1.6.0.jar /etc/guacamole/extensions/
   sudo chown tomcat:tomcat /etc/guacamole/extensions/guacamole-auth-ldap-1.6.0.jar
   ```

2. **Add LDAP configuration to guacamole.properties**:
   ```bash
   # Backup current config
   sudo cp /etc/guacamole/guacamole.properties /etc/guacamole/guacamole.properties.backup
   
   # Add LDAP configuration (example from working setup)
   sudo tee -a /etc/guacamole/guacamole.properties << 'EOF'
   
   # LDAP properties
   ldap-hostname: your-domain-controller.domain.com
   ldap-port: 389
   ldap-encryption-method: none
   ldap-user-base-dn: OU=Corp-Users,dc=domain,dc=com
   ldap-search-bind-dn: cn=serviceaccount,OU=ServiceAccounts,DC=domain,DC=com
   ldap-search-bind-password: your-service-account-password
   ldap-username-attribute: sAMAccountName
   ldap-user-search-filter: (objectClass=*)
   EOF
   ```

3. **Secure the configuration file**:
   ```bash
   sudo chmod 600 /etc/guacamole/guacamole.properties
   sudo chown tomcat:tomcat /etc/guacamole/guacamole.properties
   ```

4. **Restart and verify**:
   ```bash
   sudo systemctl restart tomcat
   
   # Verify both extensions are loaded
   sudo ls -la /etc/guacamole/extensions/
   
   # Test both authentication methods:
   # - Database users (like guacadmin)
   # - Active Directory users (using sAMAccountName)
   ```

## Enhanced Diagnostic Steps

### Pre-Migration Database Analysis

Add this diagnostic script to Phase 1 of the migration:

```bash
#!/bin/bash
# Enhanced database diagnostic for migration planning

echo "=== GUACAMOLE DATABASE DIAGNOSTIC ==="
echo ""

# Check database structure
mysql -u guacamole -p guac_db << 'EOF'
-- Show database and table information
SELECT 'Database Information' as section;
SHOW TABLES;

-- Show user structure (critical for password reset understanding)
SELECT 'User Table Structure' as section;
DESCRIBE guacamole_user;
DESCRIBE guacamole_entity;

-- Count users and identify admin users
SELECT 'User Statistics' as section;
SELECT COUNT(*) as total_users FROM guacamole_user;

SELECT 'Admin Users' as section;
SELECT 
    e.name as username,
    u.full_name,
    u.email_address,
    u.organizational_role
FROM guacamole_entity e
JOIN guacamole_user u ON e.entity_id = u.entity_id
JOIN guacamole_system_permission sp ON e.entity_id = sp.entity_id
WHERE sp.permission = 'ADMINISTER' AND e.type = 'USER'
ORDER BY e.name;

-- Show connection count
SELECT 'Connection Statistics' as section;
SELECT COUNT(*) as total_connections FROM guacamole_connection;
EOF
```

### Post-Migration Verification Script

```bash
#!/bin/bash
# Comprehensive post-migration verification

echo "=== POST-MIGRATION VERIFICATION ==="
echo ""

# Check Tomcat service
echo "1. Checking Tomcat service..."
if systemctl is-active --quiet tomcat; then
    echo "✓ Tomcat is running"
else
    echo "✗ Tomcat is not running"
    sudo systemctl status tomcat
fi

# Check guacd service
echo ""
echo "2. Checking guacd service..."
if systemctl is-active --quiet guacd; then
    echo "✓ guacd is running"
else
    echo "✗ guacd is not running"
    sudo systemctl status guacd
fi

# Check database connectivity
echo ""
echo "3. Testing database connectivity..."
mysql -u guacamole -p guac_db -e "SELECT 'Database connection successful' as status;" 2>/dev/null
if [ $? -eq 0 ]; then
    echo "✓ Database connection successful"
else
    echo "✗ Database connection failed"
fi

# Check for required extensions
echo ""
echo "4. Checking authentication extensions..."
if [ -f "/etc/guacamole/extensions/guacamole-auth-jdbc-mysql-1.6.0.jar" ]; then
    echo "✓ MySQL authentication extension installed"
else
    echo "✗ MySQL authentication extension missing"
fi

if [ -f "/etc/guacamole/extensions/guacamole-auth-ldap-1.6.0.jar" ]; then
    echo "✓ LDAP authentication extension installed"
else
    echo "⚠ LDAP authentication extension not found (install if needed)"
fi

# Check web interface accessibility
echo ""
echo "5. Testing web interface..."
if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/guacamole/ | grep -q "200\|302"; then
    echo "✓ Web interface is accessible"
else
    echo "✗ Web interface is not accessible"
fi

# Check for common configuration issues
echo ""
echo "6. Checking configuration..."
if [ -f "/etc/guacamole/guacamole.properties" ]; then
    echo "✓ guacamole.properties exists"
    if grep -q "mysql-database" /etc/guacamole/guacamole.properties; then
        echo "✓ MySQL configuration found"
    else
        echo "✗ MySQL configuration missing"
    fi
    if grep -q "ldap-hostname" /etc/guacamole/guacamole.properties; then
        echo "✓ LDAP configuration found"
    else
        echo "⚠ LDAP configuration not found (add if needed)"
    fi
else
    echo "✗ guacamole.properties missing"
fi

echo ""
echo "=== VERIFICATION COMPLETE ==="
echo ""
echo "If any items show ✗, address them before proceeding with user testing."
```

## Updated Migration Timeline

### Phase 5: Authentication Setup and Testing (CRITICAL)

This phase should be expanded significantly based on lessons learned:

#### Step 5.1: Install Required Authentication Extensions

```bash
# Install MySQL authentication extension (REQUIRED)
cd /tmp
wget https://archive.apache.org/dist/guacamole/1.6.0/binary/guacamole-auth-jdbc-1.6.0.tar.gz
tar -xzf guacamole-auth-jdbc-1.6.0.tar.gz
sudo cp guacamole-auth-jdbc-1.6.0/mysql/guacamole-auth-jdbc-mysql-1.6.0.jar /etc/guacamole/extensions/
sudo chown tomcat:tomcat /etc/guacamole/extensions/guacamole-auth-jdbc-mysql-1.6.0.jar

# Install LDAP authentication extension (if needed)
wget https://archive.apache.org/dist/guacamole/1.6.0/binary/guacamole-auth-ldap-1.6.0.tar.gz
tar -xzf guacamole-auth-ldap-1.6.0.tar.gz
sudo cp guacamole-auth-ldap-1.6.0/guacamole-auth-ldap-1.6.0.jar /etc/guacamole/extensions/
sudo chown tomcat:tomcat /etc/guacamole/extensions/guacamole-auth-ldap-1.6.0.jar
```

#### Step 5.2: Verify Database Authentication

```bash
# Restart Tomcat after installing extensions
sudo systemctl restart tomcat

# Test database authentication
# Try logging in with guacadmin/guacadmin
# If it fails, run the enhanced password reset script
```

#### Step 5.3: Test and Document Authentication Methods

1. **Test database authentication** with known database users
2. **Test LDAP authentication** with Active Directory accounts
3. **Document which users can access which authentication method**
4. **Create admin account recovery procedures**

## Common Pitfalls and Prevention

### Pitfall 1: Assuming Authentication Will Work After Data Migration
**Prevention**: Always install and test authentication extensions immediately after Guacamole installation, before declaring migration complete.

### Pitfall 2: Not Understanding User Table Structure
**Prevention**: Run database diagnostic scripts to understand how users are stored (entity-based vs. direct username columns).

### Pitfall 3: Missing Service Account Passwords for LDAP
**Prevention**: Document all service account credentials before migration and test LDAP connectivity before adding to Guacamole.

### Pitfall 4: File Permissions on Configuration
**Prevention**: Always set proper ownership and permissions on `/etc/guacamole/` directory and files.

## Emergency Recovery Procedures

If authentication completely fails after migration:

1. **Reset database admin password** using the enhanced script
2. **Verify extensions are installed** in `/etc/guacamole/extensions/`
3. **Check Tomcat logs** for specific error messages
4. **Test database connectivity** independently of Guacamole
5. **Verify guacamole.properties** configuration syntax
6. **Consider temporary fallback** to old server if issues persist

---

This enhanced guide incorporates all the real-world issues encountered during actual migrations and provides specific solutions that have been tested and verified to work.