# LimeRPi2-kodi
moonlight on OpenELEC. (Kodi)

Uses [the moonlight project](https://github.com/irtimmer/moonlight-embedded).
Also posted on [the OpenELEC forums.](http://openelec.tv/forum/12-guides-tips-and-tricks/76298-how-to-setup-moonlight-on-the-raspberry-pi#137002)

Installation:
--------------
1. Connect to your Pi over SSH.

2. This version of limelight/moonlight still needs Java, so let's put it on the pi in /storage/java and test it. (Future versions will not be java anymore, so no worries.)
```
mkdir -p /storage/java
curl -L -O -s https://github.com/HazCod/LimeRPi2-kodi/blob/master/jdk-8u33-linux-arm-vfp-hflt.tar.gz?raw=true
tar xvf jdk-8u33-linux-arm-vfp-hflt.tar.gz
rm tar xvf jdk-8u33-linux-arm-vfp-hflt.tar.gz
mv jdk1.8.0_33/* /storage/java
/storage/java/bin/java -version
```

3. Create the moonlight folder.
```
mkdir -p /storage/moonlight
cd /storage/moonlight
```
3b. Download moonlight and libopus and move them to /storage/moonlight/. 
Files will be updated automatically with a script, but we need to download it manually first for the pairing.
https://github.com/irtimmer/moonlight-embedded/releases/download/v1.2.2/limelight.jar
https://github.com/irtimmer/moonlight-embedded/releases/download/v1.2.2/libopus.so


4. Pair the pi with the computer. (substitute 192.168.0.150 with the IP of your desktop)
```
/storage/java/bin/java -jar /storage/moonlight/limelight.jar pair 192.168.0.150
```

5. Run the following command to create the script to run moonlight in /storage/moonlight/moonlight.sh
Again, substitute 192.168.0.150 with the IP address of your desktop.
```
cat >/storage/moonlight/moonlight.sh <<EOL
#!/bin/sh

#### SETTINGS

IP='192.168.0.150'
STREAMSETTINGS='-720 -60fps'

#### /SETTINGS

if [ ! ping -W 1 -c 1 $IP > /dev/null 2>&1 && echo 'Online' ]; then
    echo "Could not contact $IP, turn on host PC and retry."
    kodi-send --action=notification"(PC Offline, Turn on the host PC and retry.)"
else
  if [[ ! $LD_LIBRARY_PATH == *"/storage/moonlight"* ]]; then
    echo 'Changed library path'
    export LD_LIBRARY_PATH=/storage/moonlight:$LD_LIBRARY_PATH
  fi
  
  if ! lsmod | grep "snd_bcm2835" &> /dev/null ; then
    echo 'Loaded sound module'
    modprobe snd_bcm2835
  fi
  
  echo 'Exiting Kodi..'
  systemctl stop kodi
  
  echo 'Starting moonlight..'
  /storage/java/bin/java -jar /storage/moonlight/limelight.jar stream "$IP"
  
  echo 'Finished. Firing Kodi back up..'
  systemctl start kodi
fi
EOL
```


6. Run the following command to create the script that will launch our moonlight script.
```
cat >/storage/moonlight/run.sh <<EOL
#!/bin/sh
systemd-run /storage/moonlight/moonlight.sh
EOL
```

7. Run the following command to create the update script for you.
```
cat >/storage/moonlight/update.sh <<EOL
#!/bin/bash


vercomp () {
    if [[ "$1" == "$2" ]]
    then
        return 0
    fi
    local IFS=.
    local i ver1=($1) ver2=($2)
    # fill empty fields in ver1 with zeros
    for ((i=${#ver1[@]}; i<${#ver2[@]}; i++))
    do
        ver1[i]=0
    done
    for ((i=0; i<${#ver1[@]}; i++))
    do
        if [[ -z ${ver2[i]} ]]
        then
            # fill empty fields in ver2 with zeros
            ver2[i]=0
        fi
        if ((10#${ver1[i]} > 10#${ver2[i]}))
        then
            return 1
        fi
        if ((10#${ver1[i]} < 10#${ver2[i]}))
        then
            return 2
        fi
    done
    return 0
}

downloadFile() {
    # $1 : first argument must be version
    # $2 : second argument must be file to download
    curl -L -O -s "https://github.com/irtimmer/moonlight-embedded/releases/download/v$1/$2"
}

updateMoonlight() {
    FILE=version
    releases=$(curl --silent -L https://github.com/irtimmer/moonlight-embedded/releases/latest)
    current_version=$(if [ -f "$FILE" ]; then cat $FILE; fi)
    latest_version=$(echo "$releases" | egrep -o "/releases/download/v([0-9]\.*)+/" | egrep -o "v([0-9]\.*)+" | cut -c 2- | head -n 1)

    v_cmp=$(vercomp "$latest_version" "$current_version")

    if [ ! -f "$FILE" ] || [ "$v_cmp" == "1" ]; then
        echo "Updating moonlight to $latest_version"
        downloadFile "$latest_version" libopus.so
        downloadFile "$latest_version" limelight.jar
        echo "$latest_version" > "$FILE"
        return 0
    else
        echo "No update necessary, at latest version. ($latest_version)"
        return 1
    fi
}

updateMoonlight

EOL
```

8. And finally, make the scripts we just created executable.
```
chmod +x /storage/moonlight/run.sh
chmod +x /storage/moonlight/moonlight.sh
chmod +x /storage/moonlight/update.sh
```
9. Now run `/storage/moonlight/update.sh` to download the latest files.

Finished! Whenever you want to play games using moonlight, just run /storage/moonlight/run.sh.
You can run this using an addon such as Advanced Launcher.

To update moonlight, just run /storage/moonlight/update.sh
