# AWS EC2 SSH Connectivity Troubleshooting Guide
## GCP to AWS Migration

---

## Overview
This guide helps troubleshoot SSH connectivity issues when migrating instances from GCP to AWS. Follow these steps sequentially to resolve common connection problems.

---

## Prerequisites
- AWS Console access
- EC2 instance running
- SSH key pair (.pem file)

---

## Step 1: Configure Network Basics

### 1.1 Assign Elastic IP
1. Navigate to **EC2 Dashboard** → **Elastic IPs**
2. Click **Allocate Elastic IP address**
3. Select the newly allocated Elastic IP
4. Click **Actions** → **Associate Elastic IP address**
5. Select your instance and associate the IP

### 1.2 Configure Security Group
1. Go to **EC2 Dashboard** → **Security Groups**
2. Select the security group attached to your instance
3. Click **Edit inbound rules**
4. Add/verify the following rule:
   ```
   Type: SSH
   Protocol: TCP
   Port: 22
   Source: 0.0.0.0/0 (or your specific IP for better security)
   ```
5. Click **Save rules**

---

## Step 2: Verify Network Configuration

### 2.1 Check Internet Gateway (IGW)
1. Go to **VPC Dashboard** → **Internet Gateways**
2. Verify that an IGW is attached to your VPC
3. If not attached:
   - Select the IGW
   - Click **Actions** → **Attach to VPC**
   - Select your VPC

### 2.2 Verify Route Table
1. Go to **VPC Dashboard** → **Route Tables**
2. Select the route table associated with your subnet
3. Check the **Routes** tab
4. Ensure you have the following route:
   ```
   Destination: 0.0.0.0/0
   Target: igw-xxxxxxxx (your Internet Gateway)
   ```
5. If missing, add the route:
   - Click **Edit routes** → **Add route**
   - Destination: `0.0.0.0/0`
   - Target: Select your Internet Gateway
   - Click **Save changes**

---

## Step 3: Access Instance via EC2 Serial Console

When SSH is not working, use EC2 Serial Console for direct access.

### 3.1 Create Admin User (if needed)
Before you can use serial console, you need a user with password authentication:

1. **If you still have some access to the instance** (through existing SSH or other means):
   ```bash
   # Create a new user
   sudo adduser adminuser
   
   # Set a strong password when prompted
   
   # Add user to sudo group
   sudo usermod -aG sudo adminuser
   
   # Or add to wheel group (depending on OS)
   sudo usermod -aG wheel adminuser
   ```

2. **Enable password authentication temporarily** (for serial console):
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   
   Set:
   ```
   PasswordAuthentication yes
   ```
   
   Restart SSH:
   ```bash
   sudo systemctl restart ssh
   # or
   sudo systemctl restart sshd
   ```

### 3.2 Enable and Use EC2 Serial Console
1. Go to **EC2 Dashboard** → **Instances**
2. Select your instance
3. Click **Actions** → **Monitor and troubleshoot** → **EC2 Serial Console**
4. If not enabled, follow the prompts to enable it for your account
5. Login with the admin user you created
6. Press **Enter** a few times to get the login prompt

---

## Step 4: Fix SSH Configuration

Once connected via serial console, perform these steps:

### 4.1 Create/Fix .ssh Directory and authorized_keys

```bash
# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh

# Set correct permissions for .ssh directory
chmod 700 ~/.ssh

# Create or edit authorized_keys file
vi ~/.ssh/authorized_keys
```

**In the vi editor:**
- Press `i` to enter insert mode
- Paste your public SSH key (from your local machine: `cat ~/.ssh/id_rsa.pub`)
- Press `ESC` then type `:wq` and press `Enter` to save and exit

```bash
# Set correct permissions for authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Set correct ownership (replace 'username' with your actual username)
sudo chown -R $USER:$USER ~/.ssh
```

### 4.2 Configure SSH Daemon

```bash
# Open SSH configuration file
sudo nano /etc/ssh/sshd_config
```

**Verify/Add these critical lines:**

```
Port 22
PermitRootLogin no
PasswordAuthentication yes    # Set to 'no' after SSH key authentication works
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

**Save the file:**
- Press `CTRL + X`
- Press `Y` to confirm
- Press `Enter` to save

### 4.3 Restart SSH Service

```bash
# For Ubuntu/Debian
sudo systemctl restart ssh

# For Amazon Linux/RHEL/CentOS
sudo systemctl restart sshd

# Verify SSH service is running
sudo systemctl status ssh
# or
sudo systemctl status sshd
```

---

## Step 5: Test SSH Connection

From your local machine:

```bash
# Test connection
ssh -i /path/to/your-key.pem username@your-elastic-ip

# For Ubuntu instances
ssh -i /path/to/your-key.pem ubuntu@your-elastic-ip

# For Amazon Linux instances
ssh -i /path/to/your-key.pem ec2-user@your-elastic-ip

# Verbose mode for debugging
ssh -vvv -i /path/to/your-key.pem username@your-elastic-ip
```

---

## Common Issues and Solutions

### Issue 1: "Permission denied (publickey)"
**Solution:**
- Verify authorized_keys file permissions (should be 600)
- Verify .ssh directory permissions (should be 700)
- Ensure `PubkeyAuthentication yes` in sshd_config
- Check that the correct public key is in authorized_keys

### Issue 2: "Connection timed out"
**Solution:**
- Verify Security Group allows port 22 from your IP
- Check if Elastic IP is properly associated
- Verify IGW and route table configuration
- Check Network ACLs (NACLs) for blocking rules

### Issue 3: "Connection refused"
**Solution:**
- SSH service might not be running: `sudo systemctl start ssh`
- Check if SSH is listening: `sudo netstat -tlnp | grep :22`
- Verify firewall rules: `sudo ufw status` (if UFW is enabled)

### Issue 4: Wrong permissions on key file
**Solution:**
```bash
chmod 400 /path/to/your-key.pem
```

---

## Security Best Practices

1. **After SSH is working:**
   - Set `PasswordAuthentication no` in sshd_config
   - Restart SSH service
   - Use security groups to restrict SSH access to specific IPs

2. **Key Management:**
   ```bash
   # Generate new SSH key pair if needed
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```

3. **Restrict Security Group:**
   ```
   Source: Your-IP/32 (instead of 0.0.0.0/0)
   ```

4. **Consider using AWS Systems Manager Session Manager** as an alternative to SSH

---

## Quick Reference Commands

```bash
# Check SSH service status
sudo systemctl status sshd

# View SSH logs
sudo tail -f /var/log/auth.log          # Ubuntu/Debian
sudo tail -f /var/log/secure            # RHEL/CentOS/Amazon Linux

# Test SSH configuration
sudo sshd -t

# View current SSH connections
who

# Check listening ports
sudo netstat -tlnp | grep :22
# or
sudo ss -tlnp | grep :22
```

---

## Troubleshooting Checklist

- [ ] Elastic IP assigned and associated
- [ ] Security Group allows inbound SSH (port 22)
- [ ] Internet Gateway attached to VPC
- [ ] Route table has 0.0.0.0/0 → IGW route
- [ ] Subnet is associated with correct route table
- [ ] .ssh directory exists with 700 permissions
- [ ] authorized_keys file exists with 600 permissions
- [ ] Correct public key in authorized_keys
- [ ] PubkeyAuthentication yes in sshd_config
- [ ] SSH service is running
- [ ] Local SSH key file has 400/600 permissions
- [ ] Using correct username (ubuntu, ec2-user, admin, etc.)

---

## Additional Resources

- [AWS EC2 Key Pairs Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
- [AWS Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
- [EC2 Serial Console](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-serial-console.html)

---

**Document Version:** 1.0  
**Last Updated:** May 2026  
**Maintained by:** DevOps Team
