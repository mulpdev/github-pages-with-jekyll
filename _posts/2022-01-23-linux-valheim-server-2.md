---
title: Setting up a Linux Valheim Server 2
published: true
---

I'm using Ubuntu 20.04.3 x86_64

# Setting up a Linux Valheim Server 2: Mods

These mods are all BepInEx based.

## Version dependencies

Using a Valheim mod site (Thunderstore or NexusMods), ensure you know what dependencies are required.

In my case I had the following mods (as of 2022-01-23)

| Mod | BepInEx | Dependencies
| :--- | :--- |
| ValheimPlus 9.9.1 | denikson-BepInExPack_Valheim 5.4.900 | |
| RockerKitten-BoneAppetit-3.0.4 | denikson-BepInExPack_Valheim 5.4.1600 | ValheimModding-Jotunn-2.4.8 |
| ValheimModding-Jotunn-2.4.8 | denikson-BepInExPack_Valheim 5.4.1700 | 	ValheimModding-HookGenPatcher-0.0.3 |
| ValheimModding-HookGenPatcher-0.0.3 | denikson-BepInExPack_Valheim 5.4.1502  | |
| AnyPortal | denikson-BepInExPack_Valheim 5.4.701 | |
| Heightmap Unlimited Remake | denikson-BepInExPack_Valheim 5.4.1700 | |
| Discord Notifier | denikson-BepInExPack_Valheim 5.4.600 | |
| Valheim WebMap | Any | |

All mods are denikson-BepInExPack_Valheim 5.4.x. The tertiary version increments typically don't break anything, so we can likely use the highest version required for all of the mods. In this case it's denikson-BepInExPack_Valheim 5.4.1700

## Install a fresh Valheim server in a new directory

```
steamcmd +force_install_dir $INSTALL_DIR +login anonymous +app_update 896660 validate +exit
```

## Installing Mods

If you have a known good procedure, use that.

Otherwise, pick atarget mod and start as low in the dependency chain as possible and work your way up, using a fresh install each time. Test by enabling the install mods (verify versions!) in your mod manager on the client, and test cconnectivity to the server

So to test BoneAppetit I would do the following: 
- Install and test just BepInEx (e.g., latest)
- Install above and test next in chain (e.g., HookGenPatcher and Jotunn)
- Install above and test next in chain (e.g., BoneAppetit)

Then pick a new target app and repeat.

To combine all mods together
- Enable a mod with it's deps
- Test
- Enable another mod with deps
- Test
- Add another mod with dep etc.

### Valheim Plus

Valheim Plus packages it's own BepInEx dependency, which was lower than some of the other mods required. Therefore Valheim Plus should be installed BEFORE any BepInEx updates.

```
unzip UnixServer.zip  -d $INSTALL_DIR
```

### BepInEx

Update BepInEx to the version we want. This WILL overwrite any files modified by Valheim Plus, like `start_server_bepinex.sh`. Make a backup and merge or restore after the update.

```
unzip denikson-BepInExPack_Valheim-5.4.1700.zip
rsync -rv BepInExPack_Valheim/ $INSTALL_DIR
chmod u+x $INSTALL_DIR/start_server_bepinex.sh
cp -f ~/works20220123/start_server_bepinex.sh $INSTALL_DIR
rm -rf  BepInExPack_Valheim/ icon.png  manifest.json README.md
```

### Remaining client and server mods

Ensure that BepInEx is installed since everything needs it.

These all require a client side matching version to be installed, typically through a mod manager

Unzip the mod, move the relevant files, delete the [package files](https://valheim.thunderstore.io/package/create/docs/) to clean up the dir, repeat.

```
unzip sweetgiorni-AnyPortal-1.0.4.zip
mv AnyPortal.dll $INSTALL_DIR/BepInEx/plugins/
rm -rf icon.png manifest.json README.md
```

### Server side only mods

These are mods that do NOT also require a client side matching version to be installed. These should not cause any issues with players (unless they crash the server or something)

Unzip the mod, move the relevant files, delete the [package files](https://valheim.thunderstore.io/package/create/docs/) to clean up the dir, repeat.

```
unzip DiscordNotifier.zip
mv DiscordNotifier/ $INSTALL_DIR/BepInEx/plugins/
rm -rf icon.png manifest.json README.md
```
