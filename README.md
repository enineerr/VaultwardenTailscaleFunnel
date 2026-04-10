# Table of Contents

# Description
## What is a password manager
Password managers are tools to store user credentials and keys. Many options are widely used, albeit likely poor choices. They allow for users to have significantly more complex passwords while removing the need to memorize all of them, as well as bypassing keylogging.

A common example are in-browser password managers. All major browsers implement them, however they are ripe for credential stealers to easily grab their databases since they are often [only secured by OS user access control](https://courses.csail.mit.edu/6.857/2020/projects/6-Vadari-Maccow-Lin-Baral.pdf).
## Why self host a password manager?
Password managers which are not hosted by you, are not hosted by you. This means you do not control your data, and cannot properly audit, nor expect your data to **stay yours**. This, of course, is extremely dangerous when working with authentication credentials.

Google's password manager for example, [is not implemented with zerotrust paradigms](https://www.researchgate.net/publication/390191937_Security_Evaluation_of_Password_Managers_A_Comparative_Analysis_and_Penetration_Testing_of_Existing_Solutions). This means Google could easily read your passwords if they were so inclined, which also includes any persons who gain access to these databases. 
## What is Vaultwarden?
[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is a community-made, open-source re-implementation of Bitwarden's self-host option, in rust (of course). It is interoperable with the official Bitwarden clients, and uses substantially fewer resources than the official server.
## What is Tailscale?
[Tailscale](https://tailscale.com) is a zero-trust VPN platform that offers many useful services on top of its VPN base. It can punch through most NAT configurations using Tailscale's handshake servers which negotiate a direct connection between clients.

This means you do not need a static IP, and CG-NATs, which are in use for most IPv4 service, do not prevent you from hosting services yourself through Tailscale.
## Why use Docker?
Docker allows for containerization of services in a way which are declarative and repeatable as each instance is started from a read-only "image".
- Docker creates a virtual filesystem
- Docker sets up virtual networking
- Isolates most devices away from the container
- Docker (on linux) uses LXC, which allows processes to run on the real hardware kernel, while other parts are virtualized (ex. networking and filesystems)

Even if another container gains root privileges within the container, it cannot affect another container.

Ignoring the security benefits, Docker also makes deployment of containers trivial, and ensures sources are verified to be who they claim to be.
## What is this repository?
This repository is a simple to follow base, and tutorial to setup a self-hosted Vaultwarden instance, connected to Tailscale, within a docker environment on basically any x86-64 docker host (Windows and MacOS are not tested, but should work).

It implements Tailscale's reverse proxy service "Funnel" which allows any local service behind almost any NAT, with or without a public IPv4 address, to be accessible publicly. This is especially helpful as it allows services to be available without the need for constant connection to a VPN, which can be very inconsistent. ([Apple, please make the iOS VPN system functional](https://www.michaelhorowitz.com/VPNs.on.iOS.are.scam.php))

Funnel also provides a simple, and secure process to get HTTPs certificates for your local services. 
> \[!NOTE]
> This is especially valuable for Vaultwarden as [modern browsers will not allow cryptographic operations outside of HTTPs connections or "Secure Contexts"](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto), which would limit use to only Bitwarden official clients.

Vaultwarden also explicitly recommends [against using it's own built in HTTPs support, as it is not security-verified, and has bugs](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS). Therefore, recommends using a reverse proxy (such as Tailscale Funnel).

## Access Flowchart
![](/Images/SecureFlow.png)
# Getting Started
> \[!NOTE]
> Tailscale Funnel does not allow anything more than web traffic -- VaultWarden's main purpose (password manager) falls under this, however, its file sharing feature (Send) does not. It will not work over Funnel, and could result in the termination of your Funnel, or even Tailscale account.

> \[!NOTE]
> This guide does **NOT** include **ANY** form of backup solution. It is not a question of if you should setup backups, its a requirement. Always follow [3-2-1 backups](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/), especially for something as sensitive as a password database.
## Preparing Tailscale
### On boarding
- Go to [Tailscale.com/start](https://login.tailscale.com/start) and choose an identity provider you will continue to have access to without a password manager
- Fillout the welcome survey (use case of Personal or At-Home Use)
- Select **"Skip this introduction"** on the "Let's add your first device" screen
### Setting up HTTPs
- Go to the **DNS** tab
- Click **Manage** next to HTTPs
- Scroll to the bottom, and select **Enable HTTPS...**, then click enable
### OPTIONAL: Rename Tailnet
- Scroll to top of **DNS** page
- Click **Rename Tailnet**
- Re-roll until you find a memorable random name
### Create ACL tag
- Go to the **Access Controls** tab
- Select **Tags** in the sub tab menu
- Click **+ Create tag**
- Enter "vw" as the name, and choose "autogroup:member" as the owner
- Click **Save Tag**
### Set node attributes
- Select **Node Attributes** in the sub tab menu
- Click **+ Add node attribute**
- Add "tag:vw" to the **Targets** field
- Add "funnel" to the **Attributes** field
- Click **Save node attribute**
### Generating an OAUTH credential
- Go to the **Settings** tab
- Click **Trust credentials** in the side list
- Click **+ Credential**
- Enter a description (ex. Vaultwarden Container) and continue
- Expand the **Keys** drop down
- Select the **Write** checkbox for Auth Keys
- Add the tag "tag:vw"
- Click **Generate Credential**
- Copy down the client secret to a secure location to be deleted later - This will allow the Tailscale container to authenticate with your Tailnet
## Docker containers
This guide has been tested using Alpine Linux v3.23
### Pre-setup
Ensure you have Docker, and the compose extension installed on your host. Docker has guides available for most distros as well as MacOS and Windows.
- https://docs.docker.com/engine/install/
- https://docs.docker.com/compose/install/
### Compose
Copy this compose to a `compose.yaml`.
```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest-alpine
    container_name: vw
    restart: unless-stopped
    network_mode: service:vwts
    environment:
      # Set to false after wanted users are created
      - SIGNUPS_ALLOWED=true

      # Uncomment, and fill data to enable Push notifications
      # - PUSH_ENABLED=true
      # - PUSH_INSTALLATION_ID=
      # - PUSH_INSTALLATION_KEY=

      # Set value, uncomment, restart, and use value as password for admin panel.
      # It is recommended to keep the admin panel disabled unless needed.
      # - ADMIN_TOKEN=
      # To properly disable, comment the token, and
      # remove "admin_token" key from ./VaultWarden/config.json

      # This is needed for tailscale to operate properly
      - IP_HEADER=X-Forwarded-For
    volumes:
      - ./VaultWarden/:/data/
  vwts:
    image: tailscale/tailscale:latest
    container_name: ts
    restart: unless-stopped
    environment:
      # Place your TS auth code replacing "<>"
      - "TS_AUTHKEY=<Tailscale OAUTH secret>?ephemeral=false&preauthorized=true"
      - TS_HOSTNAME=vw
      - "TS_EXTRA_ARGS=--advertise-tags=tag:vw"
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_ACCEPT_DNS=y
    cap_add:
      - net_admin
    volumes:
      - ./Tailscale/config:/config
      - ./Tailscale/tailscale:/var/lib/tailscale
```
Set the **\<Tailscale OAUTH secret\>** to the secret obtained in Generating an OAUTH credential
### Funnel config
Create the directory `./Tailscale/config/`
Paste the following JSON into `./Tailscale/config/serve.json`
```json
{
    "TCP": {
      "8443": {
        "HTTPS": true
      }
    },
    "Web": {
      "${TS_CERT_DOMAIN}:8443": {
        "Handlers": {
          "/": {
            "Proxy": "http://127.0.0.1:80"
          }
        }
      }
    },
    "AllowFunnel": {
      "${TS_CERT_DOMAIN}:8443": true
    }
}
```
### Initialize everything in preparation for HTTPs certificates
- Run `# docker compose up -d`
This will create all necessary information for the next part by initializing the vw node in Tailscale.
### Setup Tailscale HTTPs certificates
The recommended way to is to prepare a simple script to manage this for you.
HTTPs certificates through Tailscale need to be re-made every 90 days. This can be automated if wanted, however this guide will only provide the means to create the script.
#### Getting your Tailscale Domain
- Go to your [Tailscale admin console](https://login.tailscale.com/admin/machines)
- Click on your `vw` node
- Scroll down to **TLS CERTIFICATE**
- Note down your "Domain" -- It should look something like vw.(Tailnet name).ts.net
#### Building your certificate script
Create a file, copy the following line of script, and replace \<Tailscale Domain\> with the domain you noted from the admin panel
```sh
sudo docker compose exec vwts tailscale cert "<Tailscale Domain>"
```
Then make it executable
`$ chmod +x <certs.sh>`

The script can then be ran with the docker container started
(`# docker compose up -d`)
The script can take up to 30 seconds to execute, but once finished, should have updated the **TLS CERTIFICATE** section on your Tailscale admin console to say "Status: Valid until 3 months from now"

At this stage you should have a fully functioning VaultWarden instance available at `vw.<tailnet>.ts.net:8443`, with new signups enabled. This same hostname can be used to connect to the instance using official Bitwarden clients -- on login press the server selector at the bottom (where it says **Accessing: bitwarden.com**) and choose **self-hosted**, and paste the hostname.
### Disabling (or enabling) account creation on VaultWarden
For security, after you have created the accounts you would like on your instance, you should disable account creation.

This is done through the environment variables for the VaultWarden docker container, which are in the docker compose.yaml file.
```yaml
...
  environment:
	- SIGNUPS_ALLOWED=false
...
```
After changing this setting, you need to restart the container fully.
`# docker compose down vaultwarden`
`# docker compose up -d vaultwarden`

To allow account creation again, simply follow the same steps, but set `false` to `true`
# Extras
## Changing the funnel port
[Tailscale offers 3 Funnel ports: 443, 8443, and 10000](https://tailscale.com/docs/features/tailscale-funnel#requirements-and-limitations)
8443 was chosen for this as it allows for another service to take the default HTTPs port on the same node. In the case this is not valuable, you can change the Funnel port by changing all instances of `8443` in the serve.json to one of the above 3
## Bitwarden push notifications
VaultWarden offers a quality [wiki entry](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification) on this in case this guide is not clear.

Enabling push notifications allows for automated syncing for Bitwarden clients, specifically mobile clients.

- Go to https://bitwarden.com/host/
- Provide an email address as the "Admin Email Address"
- Select your region -- With the provided compose.yaml, the US region should be chosen
- Copy down the ID and the Key
- Uncomment the Push Notifications portion of your compose.yaml
```yaml
	# Uncomment, and fill data to enable Push Notifications
      - PUSH_ENABLED=true
      - PUSH_INSTALLATION_ID=xxxxxxxxxxxxxxxxx
      - PUSH_INSTALLATION_KEY=xxxxxxxxxxxxxxxxx
```
- Populate the ID and Key variables
- Restart your VaultWarden container
`# docker compose down vaultwarden`
`# docker compose up -d vaultwarden`
- Finally, logout of all Bitwarden clients, and login again
## Backups
VaultWarden also offers a few [different ways](https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault#docker-based-automated-backups-example) to backup your database.
It is recommended to backup your compose.yaml, however unlike the VaultWarden database, the secret credentials are not encrypted.

> \[!NOTE]
> The secrets in your compose.yaml are enough to allow a threat actor to Man In The Middle attack you, and collect your account credentials. Do not provide these secrets to anyone, and ensure they are securely stored.

One (extensive) option which ensures the 3-2-1 paradigm is to:
- Stop the container `# docker compose down vaultwarden`
- Use a backup tool (ex. [restic](https://restic.net/)) and backup the VaultWarden folder locally, or even the whole directory also containing your compose.yaml
- Start the container again `# docker compose up -d vaultwarden`
- Archive the local backup
- Use `rclone` to sync the archive to a cloud provider such as Google Drive

Using dedicated backup tools allows for a seamless addition of archival encryption, and better management of versions.
# I just don't want to self host
Self hosting services is a pain, it requires a lot of research and sifting through documentation. Not to mention the upkeep and hardware needed.

Here are some options which are corporate hosted, but can be trusted more than most options.

- [Bitwarden](https://bitwarden.com/pricing/) - Bitwarden's freemium public instance is still backed by open source, zerotrust, and [externally audited code](https://bitwarden.com/help/is-bitwarden-audited/#third-party-security-audits), and can have many of the features and more of a self hosted instance for a small fee ([Currently $19.80 per year](https://bitwarden.com/pricing/)).
- [ProtonPass](https://proton.me/pass) - Proton's open source offering is a more user-friendly and integrated solution which is available as a freemium model ([Currently €35.88 per year](https://proton.me/pass/pricing)). ProtonPass has not been audited since 2023, but did well during their [audit](https://proton.me/blog/pass-open-source-security-audit).

- Bonus: [KeePassXC](https://keepassxc.org/) - KPXC offers entirely local, and secure password management. The database file can be safely synced over cloud file storage using native interfaces, or using tools like [SyncThing](https://syncthing.net/).
