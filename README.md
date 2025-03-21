<p align="center">
<img src="doc/plexlogo.png" height="35" style="padding-left: 25px">  
</p> 

# <img src="doc/t9_logo.png" height="25"> Tyzen9 - PLEX (PRIVATE REPO)

## Prep the config volume
If this is a new installation of Plex, then we may want to restore from a previous version.  In this case I backed up a Windows installation of Plex's config located in `%LOCALAPPDATA%\Plex Media Server` to a USB drive. I then plugged that into the new Ubuntu box and mounted it to `/media/usb1/plex`

Fire up an interactive docker container with the config volume mounted to `/config`, and the backup directory mounted to `/source`
```
docker run --rm -it \
  -v plex-config:/config \
  -v /media/usb1/plex:/source \
  busybox
```

Change to the directory in the volume that contains the Plex library
```
cd /config/Library/Application Support/Plex Media Server
```

Validate that this is the correct directory
```
$ ls -la

/config/Library/Application Support/Plex Media Server # ls -al
total 64
drwxr-xr-x   11 1000     1000          4096 Mar 21 13:53 .
drwxr-xr-x    3 1000     1000          4096 Mar 21 13:53 ..
-rw-------    1 1000     1000            42 Mar 21 13:53 .LocalAdminToken
drwxr-xr-x    6 1000     1000          4096 Mar 21 13:53 Cache
drwxr-xr-x    3 1000     1000          4096 Mar 21 13:53 Codecs
drwxr-xr-x    3 1000     1000          4096 Mar 21 13:53 Crash Reports
drwxr-xr-x    2 1000     1000          4096 Mar 21 13:53 Diagnostics
drwxr-xr-x    2 1000     1000          4096 Mar 21 13:53 Drivers
drwxr-xr-x    3 1000     1000          4096 Mar 21 13:53 Logs
drwxr-xr-x    6 1000     1000          4096 Mar 21 13:53 Plug-in Support
drwxr-xr-x    2 1000     1000          4096 Mar 21 13:53 Plug-ins
-rw-------    1 1000     1000           310 Mar 21 13:53 Preferences.xml
-rw-------    1 1000     1000         12216 Mar 21 13:53 Setup Plex.html
drwxr-xr-x    2 1000     1000          4096 Mar 21 13:53 Updates
```

Copy all the data from the backed up plex instance to the config volume
```
cp -r /source/* .
```

Change the permissions back to 1000:1000
```
chmod -R 1000:1000 /config*
```

You’ve got two paths: reclaim the current server or migrate the old server’s config fully. Since you’re migrating, Option 2 might be preferable to restore your previous setup, but let’s cover both.
Option 1: Reclaim the Current Server
If you just want to claim the existing server and set it up fresh (or if the old server’s data isn’t critical):
Get a Claim Token:
Go to https://plex.tv/claim (sign in with your Plex account).

Copy the token (e.g., claim-xxxx-yyyy-zzzz).

Stop Plex:
bash

docker-compose down

Add Claim Token to docker-compose.yml:
Edit your file:
yaml

services:
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    ports:
      - 32400:32400
    volumes:
      - plex-config:/config
      - /mnt/movies:/data/movies
    environment:
      - PUID=1000
      - PGID=1000
      - PLEX_CLAIM=claim-xxxx-yyyy-zzzz  # Add your token here
    restart: unless-stopped
volumes:
  plex-config:

Replace claim-xxxx-yyyy-zzzz with your token.

Start Plex:
bash

docker-compose up -d

The token is only needed for the first startup—it’ll claim the server and update Preferences.xml.

Verify:
Go to http://<plex-host-ip>:32400/web.

Sign in and check Settings > General and Network. If they’re there, it’s claimed.

Check logs:
bash

docker logs plex

Look for "Server claimed successfully" or similar.

Remove Token (Optional):
After claiming, edit docker-compose.yml to remove PLEX_CLAIM (it’s not needed anymore).

