# Introduction

qBittorrent has a feature-rich Web UI allowing users to control qBittorrent remotely.
`qbittorrent-nox` is a version of qBittorrent with a webUI instead of a windowed GUI.
This is ideal for headless servers without the X window system such as Ubuntu Server.

This guide will show you how to setup `qbittorrent-nox` to run as a managed background service (daemon) by setting it up as a `systemd` service.
It can then be customized like any other `systemd` service, to automatically start on boot, for instance.

For Ubuntu, it's advisable to install `qbittorrent-nox` from the official PPA to get the latest version.
Refer to https://github.com/qbittorrent/qBittorrent/wiki/Installing-qBittorrent for more information.

Side Note: these instructions are written with Ubuntu in mind but should be much the same if not exactly the same for any modern distro that uses `systemd`.
All instructions assume very basic knowledge of how to use the terminal.

# Create a separate user account (optional - you may want to do this for security depending on your setup)

Create the user that `qbittorrent-nox` will run under with:

```
sudo adduser qbtuser
```

Give it a password when prompted. You may leave every other value blank.

### Disable qbtuser account SSH login (optional)

You may also want to disable login for the account (from SSH) for security reasons. The account will still be usable locally:

```
sudo usermod -s /usr/sbin/nologin qbtuser
```

This can be reversed if necessary with the command:

```
sudo usermod -s /bin/bash qbtuser
```

# Initialise qBittorent

Before we set up `qbittorrent-nox` to run as a background service, it's advisable to run it once so that we can get some configuration out of the way such as the legal disclaimer.

First, switch to the user that will run qbittorent:

```
sudo su qbtuser
```

Then run `qbittorrent-nox`.
It will prompt you to accept the legal disclaimer.
You must agree to it in order to proceed.

Then, you should see the following information printed on your terminal:

```
******** Information ********
To control qBittorrent, access the Web UI at http://localhost:8080
The Web UI administrator user name is: admin
The Web UI administrator password is still the default one: adminadmin
This is a security risk, please consider changing your password from program preferences.
```

Now is a good time to adjust some qBittorrent settings.
Visit the URL mentioned in `To control qBittorrent, access the Web UI at...` (might be different in your case), and log in with the credentials given.
Then you can go to `Tools -> Options` to change settings such as the WebUI port.

Quit the running `qbittorrent-nox` process by pressing `Ctrl-c` on your keyboard in the terminal:

```
^CCatching signal: SIGINT
Exiting cleanly
```

You can now stop impersonating the qbittorent user by executing the `exit` command or simply pressing `Ctrl-d`.

# Setup the `systemd` service

On Ubuntu, system-wide `systemd` service definition files are located under `/etc/systemd/system/` (for other distros it might be a different directory), so we'll create the service definition file for `qbittorrent-nox` there.

Create a new file, `/etc/systemd/system/qbittorrent.service`, and edit it with the appropriate permissions and text editor of your choice, for example:

```
sudoedit /etc/systemd/system/qbittorrent.service
```

Save the file with the following contents or similar.
You may modify them as-needed to better suit your needs:

```
[Unit]
Description=qBittorrent-nox service
Documentation=man:qbittorrent-nox(1)
Wants=network-online.target
After=network-online.target nss-lookup.target

[Service]
# if you have systemd < 240 (Ubuntu 18.10 and earlier, for example), you probably want to use Type=simple instead
Type=exec
# change user as needed
User=qbtuser
# notice that no -d flag needed
ExecStart=/usr/bin/qbittorrent-nox
# uncomment this for versions of qBittorrent < 4.2.0 to set the maximum number of open files to unlimited
#LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

Then run `sudo systemctl daemon-reload` to update the service manager.

The qBittorrent service is now ready to be used. To start the service on system boot, refer to the next section.

# Controlling the service

- start the service: `sudo systemctl start qbittorrent`
- check service status: `sudo systemctl status qbittorrent`
- stop the service: `sudo systemctl stop qbittorrent`
- enable/disable it to start up on boot: `sudo systemctl enable qbittorrent`
    - this should output something like the following:
        ```
        Created symlink from /etc/systemd/system/multi-user.target.wants/qbittorrent.service to /etc/systemd/system/qbittorrent.service.
        ```
- the result of the previous command can be reverted with: `sudo systemctl disable qbittorrent`.
It simply disables automatic startup of the qBittorrent service.

Refer to the `systemd` documentation to know of more operations you can do on services.

# Logging

qBittorrent will still log most interesting stuff to its usual logging directory. In this example, this would be `/home/qbtuser/.local/share/data/qBittorrent/logs/`.

However, some output can probably still be viewed with:

```
sudo journalctl -u qbittorrent.service
```

For more information on how to use and customize `systemd` logging, refer to its documentation.

# `systemd` Service Dependencies (optional)

Let's say that you've configured `qbittorrent-nox` to download files to a directory that is in another drive, for example, mounted on `/media/user/volume`.
It's important that you edit the service created above and add some `systemd` dependencies to prevent qbittorrent from writing files on a directory that it should not, in case your drive fails to mount or is accidentally unmounted.

After you added the mount point to the `/etc/fstab`, it should have a line like this:

```
UUID=c987355d-0ddf-4dc7-bbbc-bab8989d0690 /media/volume  ext4     defaults,nofail 0       0
```

The `nofail` option prevents the system from stopping the boot process in case the drive can't mount on startup.

You should edit `/etc/systemd/system/qbittorrent.service` to add `local-fs.target` to the line `After=network-online.target` and add the line `BindsTo=media-volume.mount` to bind the qbittorrent service to the mount point that you want it to write the files.
Your service file should look like this:

```
# ... other stuff ...

[Unit]
# ...
After=network.target nss-lookup.target local-fs.target
BindsTo=media-volume.mount

# ...
```

The `media-volume.mount` is a `systemd` convention created dynamically at boot, based on the entries found in `/etc/fstab`.
Conventions such as these are used by `systemd` to define conditions around services, such as requiring a drive to be mounted before the service will start.
Refer to [Systemd.mount reference](http://man7.org/linux/man-pages/man5/systemd.mount.5.html) for further reading.
It follows a simple logic: if your drive is mounted on `/media/volume`, the unit name will be `media-volume.mount`, if it's on `/mnt/disk`, the unit will be `mnt-disk.mount`.

Using this to define the qbittorrent-nox service file, if the drive can't mount when booting or if the drive is unmounted after qbittorrent has been started, it will not allow it to start or force it to stop, preventing from writing when the drive is not ready or present.