# Dependencies for Dev environment
Docker Desktop Installed (With kubernetes enabled by enabling in docker destop) - .kube/config will point to docker-desktop context
https://www.docker.com/products/docker-desktop/ (Mac and Windows Can download)

Visual Studio Code 
https://code.visualstudio.com/Download (Mac and Windows Can download)

ADFS Server
Spin up a VM using Windows Server 2022

vboxuser
changeme

https://adfs.adfs.local/adfs/.well-known/openid-configuration <---gitlab will use
https://adfs.adfs.local/adfs/ls/IdpInitiatedSignOn.aspx <--- to test

192.168.1.50    adfs.adfs.local <--- Manually resolve this in etc/hosts> <--- since we host the static ip>

Set-AdfsProperties -EnableIdpInitiatedSignonPage $true <--- use powershell when you install adfs>


🛠 Step-by-Step: Setup ADFS in a Single Windows Server 2022 VM
1️⃣ Set a Static IP (important for AD & ADFS)

Inside your VM, open Control Panel → Network and Internet → Network Connections.

Right-click your active network adapter → Properties → select Internet Protocol Version 4 (TCP/IPv4) → click Properties.

Choose Use the following IP address and fill in something like:

IP address: 192.168.1.50 (something in your LAN/VM range)

Subnet mask: 255.255.255.0

Default gateway: your router IP (e.g. 192.168.1.1)

Preferred DNS server: same as IP (192.168.1.50, once we install AD DS)

Click OK and close.

👉 This makes your server reachable with a fixed IP.

2️⃣ Rename the Server

Open Server Manager (should start automatically).

Click Local Server (left sidebar).

Find Computer name → click the blue link → change it to ADFS-SERVER.

Reboot the VM when prompted.

3️⃣ Install Active Directory Domain Services (AD DS)

Open Server Manager.

Click Manage → Add Roles and Features.

Keep clicking Next until you reach Server Roles.

Check Active Directory Domain Services → Click Add Features → Next → Install.

When finished, click the yellow triangle at the top → Promote this server to a domain controller.

Select Add a new forest, name it adfs.local.

Set a Directory Services Restore Mode (DSRM) password (write it down).

Keep defaults → Install → VM will reboot.

👉 Now this VM is a Domain Controller.

4️⃣ Create a Domain User (for testing login later)

Open Start → Windows Administrative Tools → Active Directory Users and Computers.

Expand your domain adfs.local.

Right-click Users → New → User.

Example:

User logon name: testuser

Password: Password123! (disable password change if you want for lab)

Finish.

5️⃣ Install ADFS Role

Open Server Manager → Manage → Add Roles and Features.

Next → choose Active Directory Federation Services.

Add Features → Next → Install.

After install, click Configure the federation service.

Select Create the first federation server in a federation farm.

Choose your domain admin account (the one you used to log in, e.g., Administrator@adfs.local).

Certificate: For lab, you can generate a self-signed cert:

Open PowerShell as Admin and run:

New-SelfSignedCertificate -DnsName "adfs.adfs.local" -CertStoreLocation "cert:\LocalMachine\My"


Use that certificate in the wizard.

Federation Service Name → adfs.adfs.local.

Service account → use the built-in domain admin for now.

Complete wizard → Install → Reboot if required.

6️⃣ Verify ADFS is Running

Open browser inside VM.

Visit:

https://adfs.adfs.local/adfs/ls/IdpInitiatedSignOn.aspx


If it loads a login page, ADFS is working 🎉

Login with testuser@adfs.local / Password123!.

7️⃣ (Optional) OIDC Test

Open browser:

https://adfs.adfs.local/adfs/.well-known/openid-configuration


Should return JSON with endpoints (authorization, token, jwks).

This is the URL GitLab will use later.



🔹 Step 1: Create an Application Group in ADFS

Open AD FS Management → Application Groups → Add Application Group.

Choose a template:

For GitLab OIDC, select Server application accessing a web API → Next.

Name your group: e.g., GitLab OIDC Test.

Add a Server Application:

Client Identifier: gitlab-client or can use the given guid

Redirect URI: https://gitlab.local/users/auth/openid_connect/callback

Click Next.

Configure Client Secret:

ADFS generates one → you’ll copy it into GitLab.

Review → Finish.

Now ADFS knows about GitLab as a client that can request OIDC tokens.

🔹 Step 2: Configure GitLab

In GitLab (gitlab.rb or Admin → OIDC provider), add:

gitlab_rails['omniauth_providers'] = [
  {
    "name" => "openid_connect",
    "label" => "ADFS",
    "args" => {
      "name" => "openid_connect",
      "scope" => ["openid","profile","email"],
      "response_type" => "code",
      "issuer" => "https://adfs.example.com/adfs",
      "discovery" => true,
      "client_auth_method" => "query",
      "uid_field" => "sub",
      "client_id" => "<the GUID you copied>",
      "client_secret" => "<secret from ADFS>"
    }
  }
]

issuer: points to your ADFS domain (https://adfs.adfs.local/adfs)

client_id & client_secret: match what you configured in ADFS

redirect_uri: must match exactly the URI registered in ADFS

🔹 Step 3: Test Login

Go to GitLab login → you should see “ADFS” button.

Click it → you get redirected to https://adfs.adfs.local/adfs/ls/IdpInitiatedSignOn.aspx (or directly the login page).

Enter your domain user credentials (testuser@adfs.local).

GitLab should log you in using the token returned by ADFS.

🔹 Step 4: Troubleshooting Tips

If login fails → check GitLab logs for OIDC errors.

Make sure redirect URI exactly matches what’s configured in ADFS.

For local lab/self-signed cert → you may need to trust the cert in GitLab host.

If you want, I can make you a ready-to-use step-by-step guide from “I have JSON” → “GitLab login works with ADFS”, including screenshots for all settings.

Do you want me to do that?

# Pre requisite
1. Install dependencies
2. Set up folder structure
3. Set up a ADFS Server (Simple one use chatgpt)

# Folder Structure
ansible:
  - inventory
  - playbooks
  - roles:
      - pgsql
  - ansible.cfg
volumes
  - gitlab-data
      - config (Create this empty directory and wait for docker to mount the container to your host)
      - data (Create this empty directory and wait for docker to mount the container to your host)
      - logs (Create this empty directory and wait for docker to mount the container to your host)
  - gitlab-runner-config-data
  - nexus-data:
      - (Wait for docker to mount the container to your host)
  - nginx-data:
      - certs (You have to generate your own certs or place your signed certs and keys)

docker-compose.yaml (By doing docker compose up -d, you will spin up a gitlab server and a nexus server with a nginx server but please fulfil the dependencies for dev environment first)





#How to create your own SSL Cert


#NGINX-data, Add in your own certs and key. And also in your trust store, although it is self-sign hence it will still be insecure.


#NGINX configuration would be directly in the volume which mounts the