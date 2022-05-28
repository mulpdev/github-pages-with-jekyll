---
title:  Setting up a Linux Valheim Server 1 
published: true
---

I'm using Ubuntu 20.04.3 x86_64.

I will not be discussing port forwarding or general server security.

### Install SteamCMD
[https://developer.valvesoftware.com/wiki/SteamCMD][https://developer.valvesoftware.com/wiki/SteamCMD)]

## Installing Valheim server

The Steam app id for Valheim server is 896660

```
steam@server:~$ steamcmd +force_install_dir ~/valheimserver +login anonymous +app_update 896660 validate +exit
```

## Configuring the Valheim server

The `start_server.sh` script will be overwritten every time the game updates, it also has default values that should be updated. Instead of editing it, copy the script to another file, and edit that to run your server.

My file is called `val-start.sh`

```
#!/bin/bash

export templdpath=$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=./linux64:$LD_LIBRARY_PATH
export SteamAppId=892970


echo "Starting server PRESS CTRL-C to exit"

# Tip: Make a local copy of this script to avoid it being overwritten by steam.
# NOTE: Minimum password length is 5 characters & Password cant be in the server name.
# NOTE: You need to make sure the ports 2456-2458 is being forwarded to your server through your local router & firewall.
#./valheim_server.x86_64 -name "My server" -port 2456 -world "Dedicated" -password "secret"

# Simple
./valheim_server.x86_64  -nographics -batchmode -name "ReplaceName" -port ReplacePort -world "ReplaceWorld" -password "ReplaceePass" -public 1 

# Append the below to the server command to make a log file, make sure you make the logs directory first
# > "logs/val_"`date "+%Y%m%d-%H%M"`".log" 2>&1

export LD_LIBRARY_PATH=$templdpath
```

NOTE: The name and world CAN NOT be the same string

NOTE: If you use the default port of 2456, and you have players joining by IP, you will need to increment the port when providing the IP string (e.g., public_ip:2457). Steam will not talk on the default port. 

NOTE: [https://github.com/lloesche/valheim-server-docker/issues/22][https://github.com/lloesche/valheim-server-docker/issues/22] describes effect of `-nographics -batchmode` as a Unity specific workflow. I don't know exactly what, if any, effect is on the server. I have not profiled it.

## Running the Valheim server

Make your script executable and run it as the steam user

```
chmod u+x ./val-start.sh
./val-start.sh
```

# Making it a systemd service

As root, create `/etc/systemd/system/valheimserver.service`

```
[Unit]
Description=Valheim Server
Wants=network-online.target
After=syslog.target network.target nss-lookup.target network-online.target
[Service]
Type=simple
Restart=on-failure
RestartSec=5
StartLimitInterval=60s
StartLimitBurst=3
User=steam
Group=steam
ExecStartPre=/usr/games/steamcmd +force_install_dir /home/steam/valheimserver +login anonymous +app_update 896660 validate +exit
ExecStart=/home/steam/valheimserver/val-start.sh
ExecReload=/bin/kill -s HUP $MAINPID
KillSignal=SIGINT
WorkingDirectory=/home/steam/valheimserver
LimitNOFILE=100000
[Install]
WantedBy=multi-user.target
```

`ExecStartPre` is currently set to update Valheim every time it is run. This could also be made into it's own service for better seperation of functionality

**NOTE** Using a shell script in `ExecStartPre` for updating requires the full path to `/usr/games/steamcmd`

## Optional, setting overrides for environment variables

If you plan to place your server configs into version control, it is a good idea to seperate out your senesitive info

As root, execute `systemctl edit valheimserver`

```
[Service]
Environment="VAL_NAME=ReplaceName"
Environment="VAL_PORT=ReplacePort"
Environment="VAL_WORLD=ReplaceWorld"
Environment="VAL_PASS=ReplacePass"
Environment="VAL_PUBLIC=ReplacePublic"
```

Finally, modify your `val-start.sh` to use the new environment variables

```
#!/bin/bash

export templdpath=$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=./linux64:$LD_LIBRARY_PATH
export SteamAppId=892970


echo "Starting server PRESS CTRL-C to exit"

# Tip: Make a local copy of this script to avoid it being overwritten by steam.
# NOTE: Minimum password length is 5 characters & Password cant be in the server name.
# NOTE: You need to make sure the ports 2456-2458 i being forwarded to your server through your local router & firewall.
#./valheim_server.x86_64 -name "My server" -port 2456 -world "Dedicated" -password "secret"

# Simple
./valheim_server.x86_64  -nographics -batchmode  -name $VAL_NAME -port $VAL_PORT -world $VAL_WORLD -password $VAL_PASS -public $VAL_PUBLIC

# Append the below to the server command to make a log file, make sure you make the logs directory first
# > "logs/val_"`date "+%Y%m%d-%H%M"`".log" 2>&1

export LD_LIBRARY_PATH=$templdpath
```

## Running the server

Use the service name with the systemctl command

```
systemctl start valheimserver
systemctl status valheimserver
```
