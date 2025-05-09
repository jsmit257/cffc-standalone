## CFFC Standalone

This tutorial explains how to run a Center for Fungus Control as a standalone application. It includes eight individual services, including two databases that require one-time initialization if you want persistent storage across server restarts (which you probably do). Setup is fairly simple, and starting and stopping the application requires a single command. Backup and restore of the databases is also automated and explained toward the end of this tutorial. We've broken down the major topics into five sections:

* [Prerequisites](#prerequisites): what you need to get from the internet before you can begin, you may already have them installed on your server/workstation.

* [Installation](#installation): these tasks need to be performed once the host environment to initialize the data stores.

* [Run-time](#runtime) basically, just how to start and stop the application.

* [Usage](#usage): how to login, navigate, and so on.

* [Maintenance](#maintenance): a deeper dive into backup and restore functionality.

* [Advanced](#advanced): a deeper dive into what the services do, and tips on tuning and troubleshooting them.

The whole process should only take 15-20 minutes depending on your download speed.

### Prerequisites
*  All the services run in Docker containers, and don't require any additional software on your server/workstation. You can find more information on the [Docker site](https://docs.docker.com). UNIX/Linux and Mac users are probably already familiar with how to install `docker` and `docker-compose` using native package managers (or `homebrew`). For Windows users, or for anyone wanting the `docker-desktop` GUI tool (not required) we've included some links below:

    * [Windows](https://docs.docker.com/desktop/install/windows-install/): one installer to rule them all. Your Windows' licencing determines how much Docker can do on your machine, but a basic OEM license supports what we need.

    * [Mac](https://docs.docker.com/desktop/install/mac-install/) make sure to get the right one for your chip architecture. Finding out what that is is an exercise for the reader - for now. The salesman probably made a big deal about whether they sold you an Intel or ARM chip.

    * [Linux](https://docs.docker.com/desktop/install/linux-install/) you can skip to the bottom of the page for links to installing on several major distributions.

    * You can find plenty of alternatives to docker-desktop if you really need one by googling `docker GUI`, but again, this is not required nor even recommended for this application, and we won't ever mention the graphical tools again.

    Be sure Docker is starts when your system boots, or at least before continuing with this tutorial.

* Git is also recommended, although not necessarily required. Updates to the sevices will be pulled automatically without `git` every time you restart the application, but if you don't keep current with the [repo](https://github.com/jsmit257/cffc-standalone), then services that require, e.g. new environment variables, may stop working after fetching new images on application startup. Clone the repo like this, from a terminal opened to some user-writable directory, like $HOME or another place where you host projects:

  ```sh
  git clone https://github.com/jsmit257/cffc-standalone.git cffc-standalone
  ```
  If you choose not to install `git` you can download a zip archive with all the necessary files from [here](https://github.com/jsmit257/cffc-standalone/archive/refs/heads/master.zip). Unzip it to create a directory like this (you may want to rename the top-level directory to something shorter, it doesn't matter):
  ```
  cffc-standalone-master/ 
  ├── README.md 
  ├── docker-compose.yml 
  └── env
      └── template
  ```

### Installation
#### secrets file
This file has to be created manually and cannot be checked into the repo, otherwise anyone on the internet could get access to your database passwords and leaves the door open to mischief, or worse. Create an empty file named exactly `secrets` in the env directory, like so:
```
cffc-standalone-master/ 
├── README.md 
├── docker-compose.yml 
└── env
    ├── template
    └── secrets # <-- new file goes here
```
Open the file and insert the following contents:
```sh
# postgres root configuration
POSTGRES_PASSWORD=root

MYSQL_ROOT_PASSWORD=root

# maild configurations
# FIXME: finish this!!!
MAILD_=user/pass/keys/tokens???

# ussrv and usweb configuration
# US_MYSQL_PASSWORD="${MYSQL_ROOT_PASSWORD}" # this doesn't work, the value is empty
US_MYSQL_PASSWORD=root # ^^^ use constant values instead
US_REDIS_PASS=#currently unused
#you'll need an account with twilio for the next two values, a free account is OK, unless you plan to forget your password often and need to have it sent to your phone every time
US_SMS_ACCT_ID=
US_SMS_AUTH_TOKEN=

# cffc-api configuration
CFFC_HUAUTLA_PASSWORD=root

# # huautla-migration configuration (how does it get the passwords?)
# HUAUTLA_MIG_SOURCE_PASS=postgres
# HUAUTLA_MIG_DEST_PASS=postgres

# huautla-backup configuration
HUAUTLA_BACKUP_SOURCE_PASS=root # same value as CFFC_HUAUTLA_PASSWORD above

# huautla-restore configuration
HUAUTLA_RESTORE_DEST_PASS=root  # same value as CFFC_HUAUTLA_PASSWORD above

# us-backup configuration
USERSERVICE_BACKUP_SOURCE_PASS=root  # same value as US_MYSQL_PASSWORD above

# us-restore configuration
USERSERVICE_RESTORE_DEST_PASS=root  # same value as US_MYSQL_PASSWORD above
```
_Note:_ all these variables are declared in the `env/template` file for reference, they're all assigned empty values and should stay that way. *No* sensitive data should ever be pushed to `github` or any source control manager. Find a more secure way to store this data, like private cloud, private NAS, physical media like CD/DVD/sdcard or somewhere else that only you have access to.

_NB:_ these are the default values for installation. Please, *please* follow the instructions in [advanced](#advanced) to change these passwords from within the services immediately, update your secrets file with your new passwords and restart the whole application (see [restarting](#restarting) below) immediately to make sure they work *before* you start adding data to the system. If something goes wrong with this step, you can just delete the whole `persistence` directory tree and start over.
  
#### databases
After creating the secrets file, two databases need to be initialized before the application can fully start. We discuss how to skip this step in [advanced](#advanced), but if you do, no data (mycological or authentication/authorization) will be lost when the application stops.

* #### huautla
  This database stores all fungus related data, including vendors, image links (but not the images themselves), geneology, etc. Setup is as simple as running the following command from a terminal inside the project directory:
  ```sh
  $ docker-compose up huautla-initialize
  $ docker-compose down huautla-source # optional, but recommended
  ```
  The first command starts a seed database instance, and an instance to migrate the seed data to the hosts filesystem. Docker creates the necessary directory structure in the path `persistence/huautla/data` and the *initialize* service changes its ownership to postgres:postgres, then copies the actual data out of `huautla-source` and into new tables stored in the aforementioned `.../data` directory. In the unlikely event that permissions can't be changed, the service will fail - see [advanced](#advanced-huautla) for troubleshooting advice.

  The secind command stops the seed database, you'll never need it again. This won't get in the way if you don't run this step, and it will eventually stop when the application stops, but it's never a bad idea to stop it when the installation is done.

* #### userservice
  Here is where we store information on users: name, password, data related to login failures, cell phone and/or email for sending _forgot password_ resets, etc. For now, all it does is ensure that you are authorized to enter the site, but is likely to expand to include details about what you can do once in: e.g. view/add/remove mycology data, manage users, etc. Currently, anyone can create a new account, so the only real security is based on who has access to the network where the application is installed, so hosting the application on the cloud, for instance, is ill-advised.

  The steps for installation are similar to huautla, from a termimal/console opened in the project directory, run:
  ```sh
  $ docker-compose up us-initialize # this may fail the first time, it's ok, for now
  $ sudo chown 1001:1001 persistence/authnz/data
  $ docker-compose up us-initialize # this time it will succeed
  ```

  _Note:_ unlike huautla, this service starts as the `mysql` user, rather than root, so even though it tries to run the `chown` command, it fails if the docker daemon is running as root. We use numeric IDs for `user:group` because the percona container we're based on doesn't use the standard `969:969` found on most native installs, so `chown mysql:mysql` from the host *won't* work. Expect a fix, in time. Alternatively, `sudo chmod 755 persistence/authnz/data` would also work.

  There is no seed data for this database, therefore no source database service to bring `down` like there is for huautla. 

And that's it for installation. You won't have access to the data directories created in this section without _root_ or _postgres_/_mysql_ access, but that's OK. If you really need to poke around (*highly* unlikely), you can do so from inside a service container, or just *sudo* from the host. Backup and restore are services managed by the docker daemon and have the necessary access through `docker-compose` commands described in [maintenance](#maintenance).

### Runtime
That was all the heavy lifting; if you've made it this far, congratulate yourself. Now we're ready to start the application. 

#### starting
Like all commands, open a terminal in the project directory, and use the following command:
```sh
$ docker-compose up -d cffc-web
```
Dependencies built into the [docker-compose.yml](./docker-compose.yml) file handle starting all the other required services - databases, api servers, redis, mail daemon, web servers - in the right order.

#### status
You can check the status of the application anytime by running the following command:
```sh
$ docker-compose ps
```
which should produce output like the following:
```
NAME                         IMAGE                       COMMAND                  SERVICE    CREATED              STATUS              PORTS
cffc-standalone-cffc-api-1   jsmit257/cffc:lkg           "/cffc"                  cffc-api   About a minute ago   Up About a minute   
cffc-standalone-cffc-web-1   jsmit257/cffc-web:lkg       "/bin/sh -c '/bin/ba…"   cffc-web   About a minute ago   Up About a minute   0.0.0.0:10080->80/tcp, [::]:10080->80/tcp, 0.0.0.0:10443->443/tcp, [::]:10443->443/tcp
cffc-standalone-huautla-1    postgres:bookworm           "docker-entrypoint.s…"   huautla    About a minute ago   Up About a minute   5432/tcp
cffc-standalone-us-authn-1   redis:bookworm              "docker-entrypoint.s…"   us-authn   About a minute ago   Up About a minute   6379/tcp
cffc-standalone-us-db-1      percona:ps-8.0.36-28        "/docker-entrypoint.…"   us-db      About a minute ago   Up About a minute   3306/tcp, 33060/tcp
cffc-standalone-us-maild-1   bytemark/smtp               "docker-entrypoint.s…"   us-maild   About a minute ago   Up About a minute   25/tcp
cffc-standalone-us-srv-1     jsmit257/us-srv-mysql:lkg   "/user-service"          us-srv     About a minute ago   Up About a minute   
cffc-standalone-us-web-1     jsmit257/us-web:lkg         "/bin/sh -c '/bin/ba…"   us-web     About a minute ago   Up About a minute   80/tcp
```
Pay careful attention to the `STATUS` field, if it says something about `(Re)started...` then something is wrong. The most likely cause of errors will be a mis-configuration in the secrets file.

#### stopping
When you want to stop the application, you should bring all the serviced down at once, i.e. it's not correct to just stop the `cffc-web` service, because that won't stop all the dependent services that `cffc-web` requires to run. The correct command for this is just:
```sh
$ docker-compose down -t5
```
This will send *all* the services a shutdown command, and wait 5 seconds before forcibly killing them.

#### restarting
When we refer to restarting the application, we *don't* mean using any version of the `docker-compose restart ...` command. That would be complicated, at best, and likely to cause unexpected problems, anyway. Instead, we mean follow the instructions for [stopping](#stopping), followed by the directions for [starting](#starting). And it never hurts to check [status](#status) when you do this, assuming you've made changes that require a restart.

#### other considerations
Since these are just simple command lines that run from a terminal, they can be automated in various ways, like desktop shortcuts or scheduled actions. The processes to do these things are different for different OSes, but the important thing to remember is that the commands we've documented must be run from inside the project directory, or else use the `docker-compose -f </path/to/project/docker-compose.yml> ...` form of the command in your startup/shutdown scripts or actions. GUI tools for shortcuts and scheduled actions usually offer some version of a `start in directory` or similar, and scripts can just `cd /path/to/project`, or use the `-f` option with `docker-compose`

It's a good idea to schedule shutdown when the server/workstation is being powered down, so services have a chance to shutdown gracefully before being sent `SIGHUP`/`SIGKILL` signals from the OS and risk database corruption. While you're doing this, scheduling the application to start when the system boots may be convenient. Just note that this has to be scheduled *after* the docker daemon is up and running, preferrably as a dependency.

### Usage

### Maintenance
No updates are ever needed for the services because new versions will be looked for and downloaded when each service starts. But backups are your job. And knowing how to create and restore them may be important some day. Hopefully not, but it's always better not to risk it.

#### backup
Backups only need to be performed on the `huautla` and `userservice` services, and commands are available for each. This example shows both commands to run at once, but they can be run individually as well:
```sh
$ docker-compose up huautla-backup # to backup huautla, obviously
$ docker-compose up us-backup # and to backup userservice
```
More importantly than automating application shutdown, these should definitely be scheduled to run periodically. Choose a schedule based on how often you change data. Chances are, huautla data changes much more often than user data, and user data is probably much easier to manually restore manually, unless you have dozens of users to re-create after a database is corrupted.

In the case of either huautla or userservice, the backup archive is named with a timestamp including year, month, day, hour, minute and second. For convenience, the most recent backup is always symlinked to a file called `latest` in the `persistence/huautla/backups` or `persistence/authnz/backups` directories, respectively. Restoring a backup is described in the next section.

#### Restore
To restore the database to a previous state, run one of the following (it's unlikely you would need both, so be sure to choose the correct one):
```sh
$ RESTORE_POINT=yyyymmddThhMMssZ docker-compose up huautla-restore
# or
$ RESTORE_POINT=yyyymmddThhMMssZ docker-compose up us-restore
```
The file named by `RESTORE_POINT` is one of the files in the `persistence/huautla/backups/` or `persistence/authnz/backups/`directories. As you see from the examples, the filenames are a timestamp starting with a 4-digit year, and ending with seconds and a constant `Z` char, per the [RFC3339](https://www.rfc-editor.org/rfc/rfc3339) standard, with punctuation removed. Because the most recent backup is linked to a file called `latest`, you can use latest instead of, e.g. `20250101T122105Z`, for the file created at 12:21 and 5 seconds, Jan 1, 2025.

_NB:_ this will destroy any existing data in the chosen database without asking. You don't want this guy unless you really need this guy. However, it only changes user-tables for the selected database, as opposed to system-tables where passwords, and other global configurations are stored.

### Advanced

#### customizations

##### runtime customization
There are a few reasons to change the values in `docker-compose.yml`. Mostly around resource contention like port-forwarding. Maybe you want a different name or location for `data/` or `backups/`. Maybe you want to use a different database server or change the current server's username/password. Whatever your reasons, remember that this file was copied, *not* cloned, so if you want to pick up a newer copy, note that you'll have to merge the changes from the repo with the customizations you made locally. Once you get a working config, you should probably make a backup of the compose file.

For developers: you could run a standalone server from the `standalone/` directory in a cloned repo, but that is not perferred. That location is for testing only and the only artifact that should live there is `docker-compose.yml`. If you want to develop this project *and* run an always-live server, it's highly recommended to follow the steps in this README and make a new directory with just a copy of the `yml` file in it. Changes should be made to the copy so as not to taint the repo, unless changes were meant to head upstream.

##### look and feel
l
#### runtime service details
These are the services that need to be running for a healthy application:

##### huautla
##### us-authn
##### us-db
##### us-maild
##### us-srv
##### us-web
##### cffc-api
##### cffc-web

The other services in [docker-compose.yml](./docker-compose.yml) are concerned with initialization, backup and restore and should be pretty well described above.

#### Troubleshooting
We chose ports at random for the HTTP Server and, for some reason, the `huautla-standalone:` database service. If these port-mappings conflict with another service already running on the host, then the service with the conflict will fail to start. To fix this, just change the offending port in `docker-compose.yml` to one that isn't being used. In the case of the database, you could just delete the `ports:` object completely.

#### Further Reading
