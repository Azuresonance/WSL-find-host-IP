# WSL-find-host-IP
Your Linux subsystem might need to find the IP of your Windows host realibly.

## What it does
With WSL, your Windows host can access your Linux subsystem's network services by simply accessing localhost (`127.0.0.1`).

However, as of now, this **doesn't work the other way around**. You cannot access your Windows host from within WSL via localhost.

A common solution is taking advantage of WSL's dynamically generated `/etc/resolv.conf` file (https://github.com/Microsoft/WSL/issues/1032#issuecomment-535764958).

Unfortunately, this is a temporary unreliable solution, since your Windows host's IP is dynamic, that IP changes every time you reboot.


**This tutorial helps you set up a domain name (`windowshost`) for your Windows host, so that you can access it whenever you need to find your Windows host.**

## How it does it
This solution automatically detects changes in `/etc/resolv.conf`, grabs the new IP and writes it into your `/etc/hosts` file.

## Part 0: Requirements
I personally tested it on Windows 10 1909, and WSL2 of Ubuntu 20.04.
For other system versions it should work, but use at your own risk.

You will need to install the necessary packages:
```
sudo apt update
sudo apt install inotify-tools run-one
```

## Part 1: Create script
Save the following text as new file `/usr/local/sbin/update_hosts_domain.sh`
```
#!/usr/bin/bash
dir1=/etc/resolv.conf
dir2=/etc/hosts
host_domain_name='windowshost'
md5file=/etc/resolv.conf.md5sum

update-hosts () {
        newip=`cat ${dir1} |tail -1 | awk '{print $NF}'`
        sed --in-place "/${host_domain_name}/d" ${dir2}
        echo "${newip} ${host_domain_name}" >> ${dir2}
        md5sum ${dir1} > ${md5file}
}
if [[ $(md5sum ${dir1})!=$(cat ${md5file}) ]]; then
        update-hosts
fi
while inotifywait -qqe modify "$dir1"; do
        update-hosts
done
```

Make the script executable:
```
sudo chmod +x /usr/local/sbin/update_hosts_domain.sh
```


## Part 2: Add script to crontab
Open crontab with sudo:
```
sudo crontab -e
```
Choose your favorite text editor (*nano* is the easiest) and add the following line at the end of the file that automatically opened.
```
* * * * * run-one /usr/local/sbin/update_hosts_domain.sh
```
Save and exit (on *nano* it is Ctrl+X and press Enter)

## Part 3: Enable the cron service
Unlike normal Linux distros, on WSL, cron does not start automatically on boot.

You will need to do the following steps I found on Google (https://askubuntu.com/a/1166012)

### Step 1: Create startup script for cron
Run these commands to create a startup script for cron:
```
echo "service cron start" | sudo tee /usr/local/bin/cronstart.sh
sudo chmod +x /usr/local/bin/cronstart.sh
```

### Step 2: Make the startup script runnable without password
Run this command:
```
echo "$USER ALL=(ALL) NOPASSWD: /usr/local/bin/cronstart.sh"
```
And **copy its output**.
Run this command:
```
sudo visudo -f /etc/sudoers.d/cronstart
```
And paste the stuff you copied into the file that automatically opened.

### Step 3: Set up windows task
Within Windows, go to the search bar, find and run *Task Scheduler* (as administrator if your current account isn't administrators one).

Now, click *Task Scheduler Library* on the left and then *Create Task…* on the right to create a new task. You can use the following parameters to configure the task:

**General tab**:

*Name* the task anything you want, like WSL service cron start.

Choice the option *Run whether user is logged or not*.

Mark *Do not store password* and *Run with highest privileges*.

In the *Configure for* dropdown select *Windows 10*.

If you need to setup a task for another user click on the button *Change User or Group....*

**Triggers tab**:

Click *New…* to add a new trigger for this task.

In the *Begin the task* dropdown select *At startup*.

Within the *Advanced settings* you can check Delay task for 1 minute (or 30 seconds if you wish).

**Actions tab**:

Click *New…* to add a new action for this task.

Pick *Start a program* for the *action type* and then enter `C:\Windows\System32\wsl.exe` as the program to run.

At *Add arguments (optional)* set this: `sudo cronstart.sh`

**Condition tab**:
If you are using a laptop, and you want this to work even when you are on battery, you can uncheck the *start the task only if the computer is on ac power* option.

## Part 4:
**Done!**
You should now be able to use `windowshost` as a domain name for your Windows host from within WSL.
This should work even when you reboot, but you need to wait for a minute before it takes effect.
You can check by looking up your host's IP:
```
nslookup windowshost
```
## Appendix
For some reason you might want to `ping windowshost` to check the connection and find it not working.
This is because Windows 10 drops all incoming ICMP packets by default.
You can enable it by doing this: https://github.com/microsoft/WSL/issues/4171#issuecomment-559961027
