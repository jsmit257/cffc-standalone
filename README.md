## CFFC Standalone

This tutorial explains how to run a stand-alone instance of the HTTP Server and, optionally, though often preferred, the DB server. There's advice for scheduling starts, shutdowns and backups. The servers are lightweight and can stay running for as long as the machine is powered on. It's divided into 3 main sections:

* [Installation](#installation): these are the one-time tasks needed to get the host environment outfitted with the required software - in this case, that just means [Docker](https://docs.docker.com/) - and configuration.

* [Run-time](#runtime) commands: basically just means how and when to start/stop the service(s).

* [Maintenance](#maintenance): tasks that should be run periodically to ensure the health of a production system. Everything in run-time and maintenance can be automated on every OS, but the process is different for each.

The whole process should only take 15-20 minutes depending on your download speed.

### Installation
In a perfect world, these steps only need to be performed once in the lifetime of the host machine.
* Download and install the Docker binary for your OS/distro
  * [Windows](https://docs.docker.com/desktop/install/windows-install/): one installer to rule them all. Your Windows' licencing determines how much Docker can do on your machine, but a basic OEM license supports what we need.

  * [Mac](https://docs.docker.com/desktop/install/mac-install/) make sure to get the right one for your chip architecture. Finding out what that is is an exercise for the reader - for now. The salesman probably made a big deal about whether they sold you an Intel or ARM chip. `homebrew` in a terminal is a great alternative, just run:
    ```
    brew install docker docker-compose
    ```

  * [Linux](https://docs.docker.com/desktop/install/linux-install/): TLDR; just use something like the following (where `docker-compose` is probably a separate package, but still required):
    ```sh
    $your_package_manager $install_command_and_flags docker [docker-compose]
    ```

* Prepare an installation environment. For the rest of this documentation, the directory where the services live (i.e. "installation environment") will be referred to as `$CFFC_DIR`. So, if you want to run the servers from your home directory: `/Users/johnny` or `C:\Users\johnny`, etc, then `$CFFC_DIR` will mean `/Users/johnny/cffc`, `C:\Users\johnny\cffc`, etc, respectively. Any time this documentation mentions running a `docker-compose ...` command, it means run it from `$CFFC_DIR`

  * Create a the `$CFFC_DIR` directory/folder. It doesn't have be named `cffc`, but whatever you call it, this will be the location referred to as `$CFFC_DIR`. Your user's home directory is a good place to add a folder, but the location is entirely your choice.

  * Download [this file](https://github.com/jsmit257/centerforfunguscontrol/blob/master/standalone/docker-compose.yml) into the newly created `$CFFC_DIR`.

  * Open a terminal/command prompt and change directory to the new `cffc` location:
    
    ```
    cd "$HOME/cffc"  # Mac/*nix
    ```
    or
    ```
    cd "%HOME%\cffc" # Windows
    ```

    That's assuming you used your user's home directory as the installation root, and called the project `cffc`, but the names/locations don't have any special meaning as long as you know where they are.
* Run the migration, in the terminal you just opened, run these (2 separate) commands - copy/pasting the following 2 lines would work fine:
  ```
  docker-compose up --remove-orphans migration
  docker-compose down --remove-orphans
  ```
  The first command takes a while to download, but you only need to do it once, so be patient. Migration dumps a fair amount of output, but the important part is that it ends with a line like this:
  ```
  migration-1 exited with code 0 
  ```
  A code of `0` is good, it means no-error, and we're ready to continue to [runtime](#runtime). There should be a new `data/` directory created in `$CFFC_DIR`, probably owned by an administrator account and therefore difficult to browse and modify, but if you have admin control it's fine to change the owner and permissions of this directory.
  
  Any other return code (usually just a `1`) isn't good. You can check the section on [troubleshooting](#troubleshooting) to try to recover. If a solution isn't forthcoming and obvious, you might post an issue with details about your installation environment and the errors you enclountered.

### Runtime
Here's the moment we've all been waiting for. Once the migration (above) is completed successfully, in the same terminal window, just run:
```
docker-compose up --remove-orphans -d cffc-standalone
```

This command downloads the remaining part of the install - the HTTP Server - and starts it, connected to the new database you created with `docker-compose up migration`. The output should resemble this - nevermind if the timing differs, and the `cffc-` prefix is whatever name you gave the directory in the [install](#installation) steps:

```
✔ Container cffc-huautla-standalone-1  Running  0.0s 
✔ Container cffc-cffc-standalone-1     Started  0.3s 
 ```

If everything worked well so far, you should be able to visit the running server [here](http://localhost:8082/). There's only a minimal amount of data loaded from the migration, it's up to you to add the rest based on the vendors, substrates, strains that you deal with and create lifecycles w/events as your garden progresses.

This service also creates an empty `background/` directory in `$CFFC_DIR`. You can save a Portable Network Graphic (PNG) file named `image.png` to this directory and it will be used as a background for the UI, no restart needed, just refresh the browser.

Your `$FFC_DIR` is now fully live, and portable to any machine with `docker` tools installed. If you choose to move it to another machine, the first time you run `docker up ... cffc-standalone` will take some time downloading base images, but it's still just one command to start the server, no migration is needed, and everything you need is in the pre-installed `$CFFC_DIR`, and anywhere you copy it to.

In a pinch, this could run 100% from a thumb/flash drive. It would be slightly, probably not noticeably, slower, but the primary concern is that you need to bring the services down with:

```
docker-compose down --remove-orphans
```

before disconnecting the removable storage. If this isn't done, your data may become corrupted.

From now on, all you should need to know are the `up` and `down` commands in this section to start and stop the services, respectively. But, wouldn't it be better to automate the starts and stops? For the recreational client, to whom this README is dedicated, it's worth considering starting this service when the host machine boots, and stopping it with time to spare when the host machine shuts down. I'll let the pros explain the details in a minute, but the basics for any OS and any scheduler are these:
- the job/task working directory should be `$CFFC_DIR`
- the job/task command should be verbatim the corresponding `up` or `down` command described in this section

So now you know what to run and where/when to run it:

- [Windows](https://www.tenforums.com/tutorials/173596-how-create-task-run-app-script-logon-windows-10-a.html): don't know how much 11 changed things, but the task scheduler in the linked article has looked the same for a decade. There's plenty of resources on the net as long as you understand the when/where/what.

- [Mac](https://support.apple.com/guide/mac-help/open-items-automatically-when-you-log-in-mh15189/mac): this link is for login apps, apparently they did something apoplectic with normal `rc`-style startup.

- Linux: someone can contribute a `systemd` service if you want, we're not fond enough of it to want to muck around in it right now. In which case you're probably like Mac and stuck with desktop-login applications, meh.

Now that you're a pro at starting/stopping gracefully at appropriate times, go for extra credit and schedule periodic backups. Not that anything could ever go wrong, but maintenance is at least as important as getting it running in the first place.

### Maintenance
No updates are ever needed, but new versions will be looked for and downloaded when either service starts. But backups are your job. And knowing how to restore them may be important some day. Hopefully not.

#### Backup
Another simple operation that needs to be run from `$CFFC_DIR`:
```
docker-compose up backup
```
And you're done. The first time a backup is run, an new `backups/` directory is created on the host machine (right next to `data/` and `background/`) and each invocation writes a new backup file using the current date/time for the filename.

In Windows, the same task scheduler mentioned above can be used to run tasks periodically - say midnight every night. Everyone else has [crontab](https://www.unix.com/man-page/linux/5/crontab/). Do it now and you'll never have to do it again until you get a new computer. Obviously, this can cause issues if the service is stored on a removeable drive that may not be present, so we don't really recommend installing that way.

#### Restore
To restore the database to a previous state, run this from `$CFFC_DIR`:
```
RESTORE_POINT=yyyymmddThhMMssZ docker-compose up restore
```
NB: this will destroy any existing data in the existing database without asking. You don't want this guy unless you really need this guy.

The file named by `RESTORE_POINT` is one of the files in the `backups/` directory. For convenience, the most recent backup is linked to a file called `latest`, so you don't have to type that whole year/month/day/etc filename. Or just copy/paste from the day you want.

### Caveats

#### Changing the compose file
There are a few reasons to change the values in `docker-compose.yml`. Mostly around resource contention like port-forwarding. Maybe you want a different name or location for `data/` or `backups/`. Maybe you want to use a different database server or change the current server's username/password. Whatever your reasons, remember that this file was copied, *not* cloned, so if you want to pick up a newer copy, note that you'll have to merge the changes from the repo with the customizations you made locally. Once you get a working config, you should probably make a backup of the compose file.

#### Running from the `standalone/` directory in the repo
For developers: you could run a standalone server from the `standalone/` directory in a cloned repo, but that is not perferred. That location is for testing only and the only artifact that should live there is `docker-compose.yml`. If you want to develop this project *and* run an always-live server, it's highly recommended to follow the steps in this README and make a new directory with just a copy of the `yml` file in it. Changes should be made to the copy so as not to taint the repo, unless changes were meant to head upstream.

### Troubleshooting
We chose ports at random for the HTTP Server and, for some reason, the `huautla-standalone:` database service. If these port-mappings conflict with another service already running on the host, then the service with the conflict will fail to start. To fix this, just change the offending port in `docker-compose.yml` to one that isn't being used. In the case of the database, you could just delete the `ports:` object completely.

### Further Reading
The [docker-compose](https://github.com/jsmit257/centerforfunguscontrol/blob/master/standalone/docker-compose.yml) file has abbreviated versions of these instructions pinned to each service, so it should be the only file you need.
