# Updating Vagrant System Time After Sleep

After putting my Windows PC to sleep and then waking it up, system time of my Vagrant virtual machine (Ubuntu) stays the same as it was before sleep. The lag accumulates with every sleep.

Solution:

{{ toc }}

## Add A Task To Task Scheduler

Add a task to Windows Task Scheduler to force Vagrant to catch up after computer wakes up:

1. Open `Task Scheduler`.
2. (Optional) In left sidebar, create a folder for your tasks.
3. In right sidebar, click `Create Task ...` and configure it:

        General -> Name = Update Vagrant system time
        Triggers -> New
            Begin the task = On an event
            Log = System
            Source = Power-Troubleshooter
            Event ID = 1
        Actions -> New
            Action = Start a program
            Program/script = ssh
            Add arguments = root@192.168.10.10 "service ntp restart"
        Conditions -> Power -> Start the task only if the computer is on AC power = Unchecked    

## Vagrant Configuration

* Install `ntp` with `root` user:

        apt-get update
        apt-get install ntp

* Allow connecting from Windows as `root` user by adding your public key from your Windows user's `.ssh/id_rsa.pub` file to `/root/.ssh/authorized_keys` on Vagrant machine.

## Windows Configuration

* Make sure `ssh` command is available in Windows Shell, for instance by installing Git for Windows and allowing it to install Bash commands during setup.

---

Want to discuss? [Open an issue on GitHub](https://github.com/osmianski/blog/issues) or a pull request.