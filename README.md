<p align="center">
<img src="doc/plex_logo.jpg" height="105" style="padding-left: 25px">  
</p> 

# <img src="doc/t9_logo.png" height="25"> Tyzen9 - PLEX 

This Plex container is constructed from [linuxserver.io](https://docs.linuxserver.io/images/docker-plex/#application-setup) considered a [partner](https://forums.plex.tv/t/official-plex-media-server-docker-images-getting-started/172291) to providing this container for Plex.

This container in particular has been configured to leverage Nvidia GPU transcoding, which makes a VERY noticeable improvement in streaming content to Internet connected consumers making the sharing of family videos, and live TV lightning fast and stable. 

> [!TIP]
> If you do NOT have an Nvidia GPU then use `docker-compose_noGPU.yml`.

> [!WARNING]
> These instructions are not tailored for a Windows server.

# Nvidia GPU Transcoding
These instructions assume Plex is running on Docker Engine withing a Linux (Ubuntu) host. 

## Nvidia Driver Installation (Ubuntu Linux)
1. Identify your graphics card

```sh
$ ubuntu-drivers devices
== /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
modalias : pci:v000010DEd00001B81sv000019DAsd00001445bc03sc00i00
vendor   : NVIDIA Corporation
model    : GP104 [GeForce GTX 1070]
driver   : nvidia-driver-535-server - distro non-free
driver   : nvidia-driver-550 - distro non-free recommended
driver   : nvidia-driver-535 - distro non-free
driver   : nvidia-driver-470-server - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : nvidia-driver-570-server - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```
Your output will display available drivers, with recommendations tailored to your specific graphics card.

2. Install the nvidia drivers on Ubuntu

```sh
sudo ubuntu-drivers autoinstall
```

3. Reboot 
```sh
sudo reboot
```

4. Validate functionality of the driver with this command:

```sh
$ nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.120                Driver Version: 550.120        CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 1070        Off |   00000000:01:00.0 Off |                  N/A |
|  0%   38C    P8             13W /  151W |       5MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## Set up the Nvidia Container Toolkit
Full documentation for this is available here: (https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)

1. Configure the production repository

```sh
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

2. Update the packages list
```sh
sudo apt-get update
```

3. Install the toolkit
```sh
sudo apt-get install -y nvidia-container-toolkit
```

4. Configure the container runtime by using the `nvidia-ctk` command. The `nvidia-ctk` command modifies the `/etc/docker/daemon.json` file on the host. The file is updated so that Docker can use the NVIDIA Container Runtime.
```sh
sudo nvidia-ctk runtime configure --runtime=docker
```

5. Restart the Docker daemon
```sh
sudo systemctl restart docker
```

6. Run a sample workload using a CUDA container:
```sh
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.86.10    Driver Version: 535.86.10    CUDA Version: 12.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
| N/A   34C    P8     9W /  70W |      0MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## Configure Plex
1. Restart the plex container
```sh
docker-compose restart
```

2. Log into Plex and navigate to `Settings` -> `Transcoder` for the server instance. 
3. Click the `Show Advanced` button
4. Scroll down to the `Hardware transcoding device` and select your NVIDIA card
5. Enjoy!

# Restore Previous Plex Instance
If you are migrating from a previous Plex installation to a new installation of Plex then you will need a backup of the previous Plex settings. [This is a GREAT article to reference](https://support.plex.tv/articles/201370363-move-an-install-to-another-system/) on Plex's support page

I moved from Windows to a more stable Docker container on Ubuntu. I backed up a Windows installation of Plex's config located in `%LOCALAPPDATA%\Plex Media Server` to a USB drive. I then plugged that into the new Ubuntu box and mounted it to `/media/usb1/plex`

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

## Reclaim Plex Server
Now that the previous data is in the `plex-config` volume, you need to "reclaim" your Plex server.  

1. Stop the Plex container by running this command back in directory where you cloned this repo.

```sh
docker-compose down
```

2. Visit Plex.tv to obtain a "Claim Token" associated with your plex account. Go to https://plex.tv/claim (sign in with your Plex account). Copy the resulting token value (e.g., `claim-xxxx-yyyy-zzzz`).

3. Edit the `docker-compose.yml` file, removing the comments around the `PLEX-CLAIM` section, and adding the token value.

```yaml
      # This is needed the firstime you run the server to claim it.
      - PLEX_CLAIM=claim-xxxx-yyyy-zzzz  # add your token here
```

4. Start Plex back up

```sh
docker-compose up -d
```

The token is only needed for the first startup, it will claim the server and update Preferences.xml. To verify visit the Plex server at `http://\<plex-host-ip\>:32400/web.`

Sign in and check `Settings` > `General` and `Network`. If they’re there, it’s claimed.
Also in the logs a success message like this should appear:

```
Look for "Server claimed successfully" or similar.
```

5. After claiming, edit `docker-compose.yml` to remove PLEX_CLAIM (it’s not needed anymore).  Then restart the Plex container and the migration should be complete

