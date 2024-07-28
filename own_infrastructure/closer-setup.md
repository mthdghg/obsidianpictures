
### Recorder:

Script to record

```bash
#!/bin/bash

# Get the current date and time
DAY=$(date +%d)
MONTH=$(date +%m)
HOUR=$(date +%H)
MINUTE=$(date +%M)
YEAR=$(date +%Y)

# Check if a recording time is provided as a command-line argument
if [ -z "$1" ]; then
  RECTIME=$(echo "2 * 60 * 60" | bc)  # Default to 2 hours if not specified
else
  RECTIME=$(echo "$1 * 60" | bc)  # Convert provided time to seconds (assuming it's in minutes)
fi

# Create a directory for the recording
mkdir -p /root/records/CLOSER-$MONTH-$DAY-$YEAR
touch /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log
mv /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log.$HOUR-$MINUTE   # Start the recording with nohup
nohup sox -t alsa default "/root/records/CLOSER-$MONTH-$DAY-$YEAR/REC-$MONTH-$DAY-$YEAR-$HOUR-$MINUTE-.wav" trim 0 $RECTIME : newfile : restart > /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log 2>&1 &

# Script continues running in the background

sleep 84600 && pkill -SIGINT sox
```

safe record:
```bash
#!/bin/bash

# Function to check if the audio device is available
check_device() {
  aplay -l | grep "X1"
}

# Get the current date and time
DAY=$(date +%d)
MONTH=$(date +%m)
YEAR=$(date +%Y)
POWERLOSS="-"
RECDIR="./records/CLOSER-D-$MONTH-$DAY-$YEAR"

# Check if a recording time is provided as a command-line argument
if [ -z "$1" ]; then
  RECTIME=$(echo "2 * 60 * 60" | bc)
else
  RECTIME=$(echo "$1 * 60" | bc)
fi

# Create a directory for the recording
mkdir -p $RECDIR
touch $RECDIR/record-$POWERLOSS.log

# Start the recording with nohup
while true; do
  HOUR=$(date +%H)
  MINUTE=$(date +%M)
  nohup sox -t alsa hw:CARD=X1 "$RECDIR/REC-$MONTH-$DAY-$YEAR-$HOUR-$MINUTE-$POWERLOSS.wav" trim 0 $RECTIME : newfile : restart > $RECDIR/record-$POWERLOSS.log 2>&1 &
  # Record PID to monitor it
  RECORD_PID=$!

  # Wait for the recording to finish or be interrupted
  wait $RECORD_PID

  # Check if the audio device is still available
  while ! check_device; do
    sleep 1  # Wait before checking again
    echo "looking for a device"
  done
  POWERLOSS="-POWER-RECOVERED-$(date +%H)-$(date +%M)"
  touch $RECDIR/powerloss.log
  echo "$(date +%H)-$(date +%M)" >> $RECDIR/powerloss.log
done

touch "$RECDIR/RECORDING-STOPPED-MONTH-$(date +%m)-$(date +%d)-$(date +%H)-$(date +%M)"
pkill -SIGINT sox
```

```bash
pkill -f -SIGINT record.s
```

```bash
#!/bin/bash
DAY=$(date +%d)
MONTH=$(date +%m)
YEAR=$(date +%Y)
# Check if sox is running
if pgrep sox > /dev/null; then
  echo "\nEverything is fine. Recording in progress."
  echo ""
  echo ""
  #echo "------------"
  #cat /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log
  echo "-------------"
  echo "File list:"
  ls /root/records/CLOSER-$MONTH-$DAY-$YEAR/
else
  echo "SOX is not running. Recording was stopped."
  echo ""
  echo ""
  #echo "-------------"
  #cat /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log
  echo "-------------"
  echo "File list:"
  ls /root/records/CLOSER-$MONTH-$DAY-$YEAR/
fi
```


___

chechstatus.sh
```bash
#!/bin/bash
DAY=$(date +%d)
MONTH=$(date +%m)
YEAR=$(date +%Y)
# Check if sox is running
if pgrep sox > /dev/null; then
  echo "\nEverything is fine. Recording in progress."
  echo "\n-------------"
  cat /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log
  echo "-------------"
  ls /root/records/CLOSER-$MONTH-$DAY-$YEAR/
else
  echo "SOX is not running. Recording was stopped."
  echo "-------------"
  cat /root/records/CLOSER-$MONTH-$DAY-$YEAR/record.log
  echo "-------------"
  ls /root/records/CLOSER-$MONTH-$DAY-$YEAR/
fi
```

____

### Server:

changedriveuuid.sh

```bash
#!/bin/bash
# List devices.
echo "Here is your devices:"
sudo lsblk
sleep 1
sudo blkid
# Ask the user to enter the new UUID
echo "Enter the new UUID to be mounted for /mnt/vault:"
read new_uuid
# Backup the original /etc/fstab file
cp /etc/fstab /etc/fstab.bak
# Replace the UUID of the last line in /etc/fstab with the new UUID
sed -i '$ s/UUID=[^ ]*/UUID='"$new_uuid"'/' /etc/fstab
# Show the updated /etc/fstab file
cat /etc/fstab
```

synctocloud.sh
```bash
#!/bin/bash
rsync -rv --ignore-existing /mnt/recorder-closer/ /mnt/vault/
```

fstab
```
UUID=BE4E6A434E69F515 /mnt/vault auto defaults,nofail 0 2
//192.168.0.160/RECORDER-CLOSER /mnt/recorder-closer cifs credentials=/root/.smbcredentials,nofail 0 2
```

___

file-browser tree
```bash
closeradmin@closervault:~/file-browser$ tree
.
├── branding
│   ├── custom.css
│   └── img
│       ├── icons
│       │   ├── android-chrome-72x72.png
│       │   ├── apple-touch-icon.png
│       │   ├── browserconfig.xml
│       │   ├── favicon-16x16.png
│       │   ├── favicon-32x32.png
│       │   ├── favicon.ico
│       │   ├── mstile-150x150.png
│       │   ├── safari-pinned-tab.svg
│       │   └── site.webmanifest
│       └── logo.svg
├── database.db
├── docker-compose.yml
├── docker-compose.yml.save
├── nginx.conf
└── vault -> /mnt/vault
```

docker-compose.yml
```
version: '3'
services:
  filebrowser:
    image: filebrowser/filebrowser
    container_name: filebrowser
    volumes:
      - ./vault:/srv:rw
      - ./branding:/branding/
      - ./database.db:/database.db
    restart: always

  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf  # You'll need to create an nginx.conf file
      - ./vault:/srv:ro  # Make sure Nginx has read-only access to /srv
    restart: always
```

nginx.conf
```
events {}

http {
    server {
        listen 80;
        server_name closervault.cloud, closervault.mthdghg.fun, closervault;
        #return 301 https://#host#request_uri;
        client_max_body_size 10000m;  # Set the maximum allowed size of the client request body

        location / {
            proxy_pass http://filebrowser:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

___


Recorder:
Username: