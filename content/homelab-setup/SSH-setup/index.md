---
title: "Setting up SSH"
weight: 1
draft: false
description: "Setting up SSH in homeserver"
slug: "setting-up-ssh"
tags: ["Homelab", "SSH"]
series: ["Homelab Setup"]
series_order: 1
showDate: true
showTableOfContents: true
date: 2025-11-17
---

## Introduction

This article explains how to setup SSH in my homeserver. My homeserver runs on debian so all the instructions are for debian but can work for any linux based webserver. For installing debian to hosting the server from scratch, refer the video from the youtuber `hoff._world`[^video-source]

However, we will only cover the important part of the server setup that allows one to access the server using another machine on the local network i.e., initial configuration after installing debian and setting up SSH.

## Setting up root user on Debian (Skip if already done)

This setup is done in the Debian server we just booted. The `su` command (Substitute User) without any arguments defaults to switching to the `root` user, which requires the root user's password. If the root password was not set during the OS installation, you can set it using the `passwd` command.


### 1. Execute the `passwd` command

You must be logged in as a user with **sudo** privileges to perform this action.

1.  Run the `su` command, to login as the root user:
    ```bash
    su
    ```
    Then run:
    ```bash
    passwd
    ```

### 2. Enter and Confirm the New Password

1.  The system will prompt you to enter the new password for the `root` user:
    ```
    New password: 
    ```
2.  Type your desired strong password (it will not be displayed on the screen).
3.  The system will ask you to re-type the password to confirm:
    ```
    Retype new password: 
    ```

4.  Upon successful confirmation, you will see a message like:
    ```
    passwd: password updated successfully
    ```

### 3. Test the `su` command

1.  From your normal user account, try to switch to the root user:
    ```bash
    su
    ```
2.  You will be prompted for a password. Enter the **new root password** you just set.

    ```
    Password: 
    # (Your prompt will change to #, indicating you are now the root user)
    ```

## Installing OpenSSH

Debian server doesn't have `sudo` installed yet so we proceed with `su`. The `sudo` is covered in the [follwing section](#sudo). As a root user, run the following commands.


### 1. Install the SSH Server Package

Use the `apt` package manager to install the **OpenSSH server** package (`openssh-server`).

1.  **Update your package list:**
    ```bash
    apt update
    ```
2.  **Install the OpenSSH Server:**
    ```bash
    apt install openssh-server
    ```
    * **Note:** You will be prompted to confirm the installation and disk space usage. Type `Y` and press Enter.


### 2. Verify and Enable the Service

The installation process usually starts and enables the `sshd` (SSH Daemon) service automatically. You should confirm its status.`systemctl` is the process manager in Debian that also handles what services are run after startup.  

1.  **Check the status of the SSH service:**
    ```bash
    sudo systemctl status sshd
    ```
    * Look for `Active: active (running)` in the output. If it's running, you can connect. Otherwise, use the following command to start the service.
    ```bash
    sudo systemctl start sshd
    ```
2.  **Ensure the service starts automatically on boot** (if not already enabled):
    ```bash
    sudo systemctl enable sshd
    ```


### 3. Find Your Server's IP Address

You need the server's IP address to connect from a remote machine.

1.  **Display your network configuration:**
    ```bash
    ip a
    # OR
    hostname -I
    ```
2.  **Note the IP address** listed next to `inet` for your active network interface (e.g., `eth0` or `enpXsY`. Mine is a wireless (WiFi) interface so it is `wlp2s0` in my case).


### 4. Connect Remotely (from another computer)

From your local machine (Windows, macOS, or Linux), use the `ssh` client command to connect. Installation instructions for `openssh-server` on the client is based on the OS that you use.

```bash
ssh username@server_ip_address
# Example: ssh jsmith@192.168.1.100
```

<a id="sudo"></a>

## Set Up `sudo` for Your User

The first step after logging in via SSH is to set up `sudo` for your regular user, allowing you to run administrative commands without logging in as root.

1.  **Elevate to the root shell** using the root password you set during installation:
    ```bash
    su
    ```
2.  **Install the `sudo` package** (and optionally, your preferred text editor like `vim`):
    ```bash
    apt install sudo vim
    ```
3.  **Add your user to the `sudo` group** so they can use the command:
    ```bash
    /usr/sbin/usermod -aG sudo [username]
    ```
4.  **Re-login** exit the root user with `exit` command and test your new privileges:
    ```bash
    sudo su -
    ```
    (Enter your **user's** password this time, not the root password).


## Harden SSH Security (Optional)

To secure your server, you should switch from password authentication to public key authentication and change the default SSH port.

### 1. Prepare the Public Key

1.  On your **desktop machine**, generate an SSH key pair (if you don't already have one) using:
    ```bash
    ssh-keygen 
    ```
    I prefer a stricter key with 4096 bytes and use RSA for key generation, so the command is as follows.
    ```bash
    ssh-keygen -t rsa -b 4096
    ```
2.  Copy the content of the **public key** (the file ending in `.pub`) to your clipboard.


### 2. Edit the SSH Server Configuration

1.  On your **server**, use `sudo` to edit the SSH configuration file:
    ```bash
    sudo vim /etc/ssh/sshd_config
    ```

2.  Make the following changes:
    * **Change Port**: Change the default `Port 22` to an arbitrary port, such as **2222**. This is to prevent the attackers who scan open ports with a list of known ports, like `Port 22`.
        ```config
        Port 2222
        ```

    * **Disable Root Login**: Set `PermitRootLogin` to **no**. This disallows anyone to login as root user directly.
        ```config
        PermitRootLogin no
        ```

    * **Disable Password Authentication**: Set `PasswordAuthentication` to **no**. This forces to only use the public-private key authentication.
        ```config
        PasswordAuthentication no
        ```

3.  Save and exit the file.

### 3. Apply the Public Key and Restart

1.  **Create the `authorized_keys` file** for your user (if it doesn't exist) and paste the public key you copied from your desktop machine:
    ```bash
    mkdir ~/.ssh
    vim ~/.ssh/authorized_keys
    # Paste your public key content here
    ```

2.  **Restart the SSH service** to apply the configuration changes:
    ```bash
    sudo systemctl restart sshd
    ```

3.  **Test the new connection** from your client desktop, specifying the new port (`-p`) and your private key file (`-i`):
    ```bash
    ssh -p 2222 -i [path_to_private_key] [user]@[IP_address]
    ```


## References (Video and AI Source)



The instructions provided for setting up the Linux home server were extracted from a video tutorial[^video-source]. The transcription and summary service was performed by an AI assistant [^ai-source] and were modified as per my usage.



### Citations

[^video-source]: **Video Source:** "How to Set Up a Linux Home Server from Start to Finish!" by hoff._world. Available at: [https://youtu.be/hzpN-JhBJBQ](https://youtu.be/hzpN-JhBJBQ)

[^ai-source]: **Summary Source:** Gemini (Google's AI Assistant), extracted and formatted from the video transcript between 16:03 and 25:03.