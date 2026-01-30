## Potential method to copy over Harv files (Provided OS and Software would be reinstalled)
Looked at the following stack overflow threads: 

https://stackoverflow.com/questions/41274007/anaconda-export-environment-file   -  Exporting anaconda envs

https://askubuntu.com/a/7811 -thread about copying that also mentions rsync


### Before backup begins, Users of the HPC must be instructed to back up their Conda Envs

### Each user on the HPC must run this (not admin):

```Bash
#!/bin/bash

# 1. Create a specific folder for YAML backups
mkdir -p ./env_backups_yaml

# 2. Extract clean environment names
# Filtering out headers (#), paths (/), and empty lines
ENVS=$(conda env list | awk 'NR>2 && $1 !~ /#/ && $1 !~ /\// {print $1}')

for env in $ENVS; do
    echo "Exporting YAML for: $env"
    
    # 3. Export to YAML format
    # Using the -n flag to target the specific environment
    conda env export -n "$env" > "./env_backups_yaml/${env}_env.yml"
done

echo "Finished, Backups are in the env_backups_yaml folder."
```
### This saves all conda environments as YAML files in:
### ~/env_backups_yaml/

#### To restore, run: 
```Bash 
conda env create -f env.yml
```

### Another method is using conda list and running this (However I do not know how worth it this is and if YAML is better option as it includes PIP installs)

```bash
#!/bin/bash
# 1. Create the folder using a relative path for reliability
mkdir -p ./env_backups

# 2. Tighten the filter to skip ANY line with a # or /
ENVS=$(conda env list | awk 'NR>2 && $1 !~ /#/ && $1 !~ /\// {print $1}')

for env in $ENVS; do
    echo "Backing up: $env"
    # 3. Use the relative path to the folder we just made
    conda list -n "$env" --export > "./env_backups/${env}.txt"
done
```
#### To restore, run: 
```Bash 
# Source - https://stackoverflow.com/a
# Posted by javac, modified by community. See post 'Timeline' for change history
# Retrieved 2026-01-22, License - CC BY-SA 4.0

$ conda create -n <environment-name> --file env.txt
```


### Now the HPC backup instructions (Rsync seemed to be a common option, it preserved metadata/permissions and can resume after interupted transfers aswell as work across different servers)

#### First change permissions of all users so that they cannot make any edits/run stuff during the backup

```BASH
#!/bin/bash

# Ensure we don't lock out the person running the script
CURRENT_USER=$(whoami)

# Loop through all "human" users (UID 1000+) and 65534 ghost user? 
for user in $(awk -F: '$3 >= 1000 && $3 != 65534 {print $1}' /etc/passwd); do
    
    # Skip the current admin running the script
    if [ "$user" == "$CURRENT_USER" ]; then
        echo "Skipping you ($user)..."
        continue
    fi

    # Check if the user has sudo privileges
    if id -nG "$user" | grep -qw "sudo"; then
        echo "Skipping Admin: $user"
        continue
    fi

    echo "--- Locking & Clearing User: $user ---"

    # 1. Lock access (Password + SSH Keys via Expiry)
    sudo passwd -l "$user"
    sudo chage -E 0 "$user"

    # 2. Kill all processes for this specific user
    # -9 is 'SIGKILL' (force), -U matches the Real User ID
    sudo pkill -9 -U "$user"
    scancel -u "$user"
    
    echo "Done."
done

echo "HPC is now locked. Only Admins remain."
```

#### (If applicable) Save a small list of install files installed with apt get/os details for before/when ubuntu is freshly installed: 
```BASH
#for install files to be saved, do 
dpkg --get-selections | grep -v deinstall > ubuntu-files

#Save harvey system specs/os details using: 
#Overwrite hostnAmectl onto txt
hostnamectl >> harveyspecs.txt

# Append Disk info
echo -e "\n--- STORAGE INFO ---" >> A
lsblk >> harveyspecs.txt

# Append PCI HArdwAre info
echo -e "\n--- HARDWARE DEVICES ---" >> A
lspci >> harveyspecs.txt

# Append CPU info
echo -e "\n--- CPU INFO ---" >> A
lscpu >> harveyspecs.txt
``` 
then reinstall apt packages using: 

```BASH 
sudo dpkg --set-selections <./ubuntu-files 

apt-get -y update
apt-get dselect-upgrade
```


#### AFTER THE BACKUP, run the unlock script for the users

```BASH
#!/bin/bash

# Ensure we have the current user to skip them if necessary
CURRENT_USER=$(whoami)

echo "--- Starting HPC Unlock Process ---"

# Loop through all "human" users (UID 1000+) excluding the ghost user
for user in $(awk -F: '$3 >= 1000 && $3 != 65534 {print $1}' /etc/passwd); do
    
    # Skip the current admin running the script
    if [ "$user" == "$CURRENT_USER" ]; then
        continue
    fi

    # Skip other Admins (who were never locked)
    if id -nG "$user" | grep -qw "sudo"; then
        continue
    fi

    echo "Unlocking User: $user"

    # 1. Unlock the password
    sudo passwd -u "$user"

    # 2. Remove the account expiration date (-1 sets it to 'never')
    sudo chage -E -1 "$user"
    
    echo "Access restored for $user."
done

echo "--- All users have been unlocked. ---"

```

### Now the HPC backup instructions (Rsync seemed to be a common option, it preserved metadata/permissions and can resume after interupted transfers aswell as work across different servers)

```BASH
!/bin/bash
# ------------------------------------------------------------
# Simple HPC Backup Instructions
# ------------------------------------------------------------
# Purpose:
#   Example script showing how users and admins can back up
#   home directories and conda environments to RDS storage
#   before a system rebuild.
# ------------------------------------------------------------


#IMPORTANT: Before running command use TMUX, to make sure backup proceeds even if SSH connection drops
#name the session
tmux new -s hpc_backup
#then run the backup command and CTRL + B then d to detach from the tmux window
#to reattach to window that will then run in the background: 
tmux ls 
tmux a -t hpc_backup
#once finished, CTRL + D to exit tmux session


# Admin backs up all /home directories
# ------------------------------------------------------------
# Run this as root or with sudo:
#IMPORTANT, DO A DRYRUN OF THIS FIRST TO SEE HOW LONG EVERYTHING WOULD TAKE : sudo rsync -aAXvhR --dry-run --progress \ 
#May need to reinstall slurm using sudo apt install slurm-wlm and used the backed up slurm files in etc
#backing up /sys /usr and /run is danger territory


#RUN Backup command from root
cd /

# Mount point for RDS backup
BACKUP_DEST="/mnt/Harvey_RDS_Backup/home_backup"

sudo rsync -aAXvhR --progress \ 
  --exclude 'home/*/.cache' \
  --exclude 'home/software/' \
  --exclude=/dev/* \
  --exclude=/proc/* \
  --exclude=/sys/* \
  --exclude=/tmp/* \
  --exclude=/run/* \
  --exclude=/mnt/* \
  --exclude=/media/* \
  --exclude="swapfile" \
  --exclude=/lost+found \
  /etc \
  /home \
  /usr \
  /var \
  /opt \
  /root \
  /srv \
  "$BACKUP_DEST/"


#Backup verification (make sure file size matches at source and destination location aswell as file count)
echo "--- Verification ---"
# Check total size
du -sh /etc /home /usr /var /opt /root /srv
du -sh "$BACKUP_DEST"

# Check file counts (if these match, you're usually golden)
echo "Source file count:"
find /etc /home /usr /var /opt /root /srv -type f | wc -l
echo "Backup file count:"
find "$BACKUP_DEST" -type f | wc -l

#---------------------------------------------------------------------------------------------------------

#WHEN considering parrarelisation for speed we could use fpsync to: 
# Install fpart first if not already present
sudo apt update && sudo apt install fpart -y

# Use 24 concurrent jobs to balance metadata load and network throughput
# Use 24 workers (-n) and pass rsync options (-o) 
sudo fpsync -v -n 24 \
  -o "-aAXhR \
  --numeric-ids \
  --exclude='**/.cache' \
  --exclude='home/software/' \
  --exclude='/dev/*' \
  --exclude='/proc/*' \
  --exclude='/sys/*' \
  --exclude='/tmp/*' \
  --exclude='/run/*' \
  --exclude='/mnt/*' \
  --exclude='/media/*' \
  --exclude='swapfile' \
  --exclude='/lost+found'" \
  /etc \
  /home \
  /usr \
  /var \
  /opt \
  /root \
  /srv \
  "$BACKUP_DEST/"


#--------------------------------------------------------------------------------------------

#  Restore (after reinstall)
# ------------------------------------------------------------
# Copy back user data:
# sudo rsync -avh --progress /mnt/Harvey_RDS_Backup/home_backup/ source/
#
# Then fix ownership:
# sudo chown -R username:username /home/username

```
