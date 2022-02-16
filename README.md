# wine-podman

| License | Versioning | Build |
| ------- | ---------- | ----- |
| [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) | [![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release) | [![Build status](https://ci.appveyor.com/api/projects/status/7bdfkond46b95ysy/branch/master?svg=true)](https://ci.appveyor.com/project/nikAizuddin/wine-podman/branch/master) |

Wine with Podman.


## Building

Build image:
```
podman build -t extra2000/wine/fedora34 -f Dockerfile.fedora34 .
```

Clone WINE repository:
```
git clone -b wine-7.2 https://github.com/wine-mirror/wine.git src/wine
chcon -R -v -t container_file_t src/wine
```

Create build directories:
```
mkdir -pv ./build/wine{32,64}
chcon -R -v -t container_file_t ./build
```

Build for 64bit:
```
podman run -it --rm -v ./src/wine:/opt/src/wine:rw -v ./build/wine64:/opt/build/wine64:rw --workdir /opt/build/wine64 localhost/extra2000/wine/fedora34 bash
/opt/src/wine/configure --enable-win64
make -j16
```

Build for 32bit:
```
podman run -it --rm -v ./src/wine:/opt/src/wine:rw -v ./build/wine64:/opt/build/wine64:rw -v ./build/wine32:/opt/build/wine32:rw --workdir /opt/build/wine32 localhost/extra2000/wine/fedora34 bash
/opt/src/wine/configure --with-wine64=/opt/build/wine64
make -j16
```


## VLC Testing

On Fedora 34 and above (which uses PipeWire), install `pulseaudio-utils` and then listen for audio connection:
```
sudo dnf install pulseaudio-utils
pactl load-module module-native-protocol-tcp port=4713 listen=127.0.0.1
```

Download VLC for Windows installer and put the installer into `./wineprefix`. Then, allow the directory to be mounted into container:
```
chcon -R -v -t container_file_t ./wineprefix
```

Create SELinux Security Policy for WINE Podman:
```
cp -v selinux/wine_podman.cil{.example,}
```

Import the SELinux Security Policy:
```
sudo semodule -i selinux/wine_podman.cil /usr/share/udica/templates/base_container.cil
```

Spawn WINE Podman:
```
podman run -it -e DISPLAY=$DISPLAY --device=/dev/dri --network=host --rm -v /tmp/.X11-unix:/tmp/.X11-unix:rw -v ./src/wine:/opt/src/wine:ro -v ./build/wine64:/opt/build/wine64:ro -v ./build/wine32:/opt/build/wine32:ro -v ./wineprefix:/root:rw -e PULSE_SERVER=tcp:127.0.0.1:4713 --security-opt label=type:wine_podman.process localhost/extra2000/wine/fedora34 bash
```

Test make sure the following commands are working:
```
/opt/build/wine32/wine winecfg
/opt/build/wine32/wine taskmgr
/opt/build/wine32/wine explorer
/opt/build/wine32/wine notepad
/opt/build/wine32/wine cmd
```

Check audio connection to host:
```
pactl info
```

After finished VLC installation, start `vlc.exe` using the following command:
```
/opt/build/wine32/wine vlc.exe
```

*Note: The `vlc.exe` can be found at `"${HOME}/.wine/drive_c/Program Files/VideoLAN/VLC/vlc.exe"`.*

After finished testing, unload the `module-native-protocol-tcp` module on host:
```
pactl unload-module module-native-protocol-tcp
```
