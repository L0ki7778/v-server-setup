# VM Server Setup

Here is a setup guide for adding ssh-authentification, disabling authentification with credentials, as well as installing nginx and connecting to your personal Git-Account.
After completing this guide, you should be able to connect to a custom nginx HTML-Template.

## Table of Contents

- [1. SSH Key Generation and Server Access](#1-ssh-key-generation-and-server-access)
  - [Generate an ED25519 SSH key locally](#generate-an-ed25519-ssh-key-locally)
  - [Create SSH config](#create-ssh-config)
  - [Copy public key and add to server](#copy-public-key-and-add-to-server)
- [2. Disable Password Authentication](#2-disable-password-authentication)
- [3. Install and Configure Nginx](#3-install-and-configure-nginx)
  - [Configure Nginx to run on port 8081](#configure-nginx-to-run-on-port-8081)
- [4. Create and Enable Alternative HTML](#4-create-and-enable-alternative-html)
- [5. Connect Server to Personal Git Account](#5-connect-server-to-personal-git-account)
  - [Generate a Git-specific SSH key](#generate-a-git-specific-ssh-key)
  - [Configure Git user information](#configure-git-user-information)
  - [Add the key to your GitHub account](#add-the-key-to-your-github-account)
  - [Create Git SSH config entry](#create-git-ssh-config-entry)
  - [Test the Git connection](#test-the-git-connection)
- [Troubleshooting](#troubleshooting)
  - [Issue: `ssh-copy-id` doesn't work on Windows](#issue-ssh-copy-id-doesnt-work-on-windows)
  - [Issue: Cannot connect to Git with SSH alias in `~/.ssh/config`](#issue-cannot-connect-to-git-with-ssh-alias-in-sshconfig)

## 1. SSH Key Generation and Server Access

### Generate an ED25519 SSH key locally

On your local machine, generate a new SSH key:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/vm_server_ed25519
```

This creates a key pair at `~/.ssh/vm_server_ed25519` and `~/.ssh/vm_server_ed25519.pub`.

### Create SSH config

Create a config file in your SSH folder for easier connections:
```bash
nano ~/.ssh/config
```

Add the following configuration:
```bash
Host vm_server
    HostName <123.45.67.89>   #enter your real IP
    User <deploy_user>        #enter your username
    IdentityFile ~/.ssh/vm_server_ed25519
```

### Copy public key and add to server

Display your public key:
```bash
cat ~/.ssh/vm_server_ed25519.pub
```

Connect to the VM server using password authentication:
```bash
ssh <deploy_user@192.168.1.100>
```

Agree to the fingerprint and enter your password. Once connected to the server, append your public key to the authorized keys:
```bash
nano ~/.ssh/authorized_keys
```

Paste your public key content into the file, then save and exit (`Ctrl+X`, then `Enter`, then `Enter`).

Log out from the server:
```bash
logout
```

Verify you can now connect via SSH without a password:
```bash
ssh vm_server
```

## 2. Disable Password Authentication

Connect to your VM server:
```bash
ssh vm_server
```

Edit the SSH daemon configuration:
```bash
sudo nano /etc/ssh/sshd_config
```

Find the line `#PasswordAuthentication no` and remove the `#` to uncomment it:
```bash
PasswordAuthentication no
```

Save and exit the editor (`Ctrl+X`, then `Enter`, then `Enter`).

Check config before restarting the server
```bash
sudo sshd -t
```

Restart the SSH service to apply changes:
```bash
sudo systemctl restart ssh.service
```

Restart the SSH service to apply changes:
```bash
sudo systemctl restart ssh.service
```

Verify the service is running:
```bash
sudo systemctl status ssh.service
```

## 3. Install and Configure Nginx

Install Nginx on the server:
```bash
sudo apt update
sudo apt install nginx -y
```

Verify it's running:
```bash
sudo systemctl status nginx.service
```

### Configure Nginx to run on port 8081

Edit the Nginx configuration file:
```bash
sudo nano /etc/nginx/sites-enabled/alternatives
```

Paste the following config to the newly create file
```nginx
server {
        listen 8081;                            #listen to incoming request at port 8081 (default port: 80)
        listen [::]:8081;

        root /var/www/alternatives;             #Path to alternativ .html file
        index alternate-index.html;             #Specified which .html has to be used

        location / {
                try_files $uri $uri/ = 404;     #works like router/fallback to look for similar .html file name and returns 404 if no match
        }
}
```

Save and exit (`Ctrl+X`, then `Enter`, then `Enter`).

## 4. Create and Enable Alternative HTML

Create the directory structure for your alternative HTML:
```bash
sudo mkdir /var/www/alternatives
```

Create an alternative index file:
```bash
sudo nano /var/www/alternatives/alternate-index.html
```

Add sample content:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello, Nginx!</title>
</head>
<body>
    <h1>Hello, Nginx!</h1>
    <p>I have just configured out Nginx web server on Ubuntu Server!</p>
</body>
</html>
```

Save and exit (`Ctrl+X`, then `Enter`, then `Enter`).

Reload Nginx to apply changes:
```bash
sudo service nginx restart
```

Check nginx status(optional):
```bash
sudo systemctl status nginx.service
```

## 5. Connect Server to Personal Git Account

### Generate a Git-specific SSH key

On the server, generate a new SSH key for Git authentication:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/git_deploy_ed25519
```

Display the public key:
```bash
cat ~/.ssh/git_deploy_ed25519.pub
```

### Configure Git user information

Set your Git username and email on the server. **Make sure these match your GitHub account details:**

```bash
git config --global user.name "Your GitHub Username"
git config --global user.email "your-email@example.com"
```

Verify the configuration:
```bash
git config --global user.name
git config --global user.email
```

### Add the key to your GitHub account

1. Go to GitHub → Settings → SSH and GPP keys
2. Click "New SSH key"
3. Paste the public key content
4. Give it a descriptive name (e.g., "VM Server Deploy Key")
5. Save the key

### Create Git SSH config entry

Edit your SSH config:
```bash
nano ~/.ssh/config
```

Add a Git-specific host entry:
```bash
Host github_vm
    HostName github.com
    User git
    IdentityFile ~/.ssh/git_deploy_ed25519
```

### Test the Git connection

Start the SSH agent and add your key:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/git_deploy_ed25519
```

Test the connection:
```bash
ssh -T git@github.com
```

You should see a message confirming successful authentication:
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

## Troubleshooting

### Issue: `ssh-copy-id` doesn't work on Windows

**Problem:** The `ssh-copy-id` command is not available on Windows, and manual path copying can fail due to path mismatches (e.g., Windows paths don't follow the `/home/` structure).

**Solution:** Use `cat` and manual copy-paste instead:

On your local machine, display your public key:
```bash
cat ~/.ssh/vm_server_ed25519.pub
```

Copy the output manually, then on the server add it to `~/.ssh/authorized_keys` using the steps in Section 1.

### Issue: Cannot connect to Git with SSH alias in `~/.ssh/config`

**Problem:** After creating an SSH key alias in the config file (e.g., `Host github_vm`), you still cannot authenticate with GitHub and see "Permission denied (publickey)" errors.

**Cause:** The SSH agent is not running or the key has not been added to the authentication agent.

**Solution:** Initialize the SSH agent and add your key:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/git_deploy_ed25519
```

Verify the key was added:
```bash
ssh-add -l
```

The key should now appear in the list. Test the connection again:
```bash
ssh -T git@github.com
```

For persistent SSH agent across sessions, consider adding the agent startup to your shell profile (`.bashrc` or `.zshrc`).