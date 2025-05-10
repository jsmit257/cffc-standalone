## [CFFC Standalone](https://github.com/jsmit257/cffc-standalone)

This tutorial explains how to run a Center for Fungus Control standalone application. What we call an application is the composition of eight individual services that need to be running together. It includes two databases that require one-time initialization if you want persistent storage across server restarts (which you probably do). Setup is fairly simple, and starting and stopping the application require a single command, each. Backup and restore of the databases is also automated and explained later on in this tutorial. We've broken down the major topics into six sections:

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
At least 2 of the following 3 steps should be completed for a successful installation. The section on [ssl certificates](#ssl-certificates) can be deferred or ignored until you've successfully logged into the site.

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

_NB:_ these are the default values for installation. Please, *please* follow the instructions in [passwords](#passwords) to change authentication values from within the services immediately, then update your secrets file with your new passwords, tokens, etc and restart the whole application (see [restarting](#restarting) below) immediately to make sure they work *before* you start adding data to the system. If something goes wrong with this step, you can just delete the whole `persistence` directory tree and start over.

#### databases
After creating the secrets file, two databases need to be initialized before the application can fully start. We discuss how to skip this step in [advanced](#advanced), but if you do, no data (mycological or authentication/authorization) will be lost when the application stops.

* ##### huautla
  This database stores all fungus related data, including vendors, image links (but not the images themselves), geneology, etc. Setup is as simple as running the following command from a terminal inside the project directory:
  ```sh
  $ docker-compose up huautla-initialize
  $ docker-compose down huautla-source # optional, but recommended
  ```
  The first command starts a seed database instance, and an instance to migrate the seed data to the hosts filesystem. Docker creates the necessary directory structure in the path `persistence/huautla/data` and the *initialize* service changes its ownership to postgres:postgres, then copies the actual data out of `huautla-source` and into new tables stored in the aforementioned `.../data` directory. In the unlikely event that permissions can't be changed, the service will fail - see [advanced](#advanced-huautla) for troubleshooting advice.

  The secind command stops the seed database, you'll never need it again. This won't get in the way if you don't run this step, and it will eventually stop when the application stops, but it's never a bad idea to stop it when the installation is done.

* ##### userservice
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

#### ssl certificates
This is actually an advanced topic, and for that reason, the default configuration delegates to the internal, self-signed (and therefor insecure) certificates bundled with the `cffc-web` image. This causes browsers to present a series of warnings before allowing you to access the site. 

There's no actual danger, since _you_ are the owner of this application (i.e. what would you be stealing from yourself?), but it's a cleaner user experience to use authentic certs. See the advanced topic on [using certs](#using-certs) below for instructions to fix this.

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
Pay careful attention to the `STATUS` field, if it says something about `Restarting...` then something is wrong. The most likely cause of errors will be a mis-configuration in the secrets file.

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
If everything looks good after starting the application, then you're ready to access the site. The default configuration binds to ports 10080 and 10443 on all of the host machine's network adaptors. The only one we can recommend here is `https://localhost:10443`, since we don't know your hosts other names and/or IP addresses. Connecting to `http://localhost:10080` is supported for legacy reasons and immediately redirects to SSL port *10043*. You can, of course, substitute your host's actual name or an IP for `localhost` in the above examples, but read the section on [ssl hostname](#ssl-hostname) below to configure that properly.

When presented with the login screen, you will need to create a new user. The username must be 8 characters long, but there are currently no other restrictions. When you've entered a valid name, the add button will be visible. Click add, and enter at least one of email or cell - this is required and is used for password reset in case you forget. Passwords never expire. After entering the same password twice, click the save button to create the user. Passwords never expire, but you can change them at any time.

The main menu for the site is subtly displayed as the top row of text. There are three categories of menus, and a variable number of items for each category. The header row for _lifecycle_ and _generation_ are also menus, otherwise, tables are pertty straightforward. Buttons are labeled appropriately. A detailed understanding of how to manage the data is well beyond the scope of this document, but a good place to start reading is [here](https://jsmit257.github.io/huautla/#local-database).

### Maintenance
No updates are ever needed for the services because new versions will be looked for and downloaded when each service starts. But backups are your job. And knowing how to create and restore them may be important some day. Hopefully not, but it's always better not to risk it.

#### backup
Backups only need to be performed on the `huautla` and `userservice` services, and commands are available for each. This example shows both commands to run at once, but they can be run individually as well:
```sh
$ docker-compose up huautla-backup  # to backup huautla, obviously
$ docker-compose up us-backup       # and to backup userservice
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
The topics in this section require a little more care than what you've seen thus far in this tutorial. We recommend making small changes, just a few at a time, and testing them often before moving on to the next change.

#### passwords
##### huautla database
##### userservice database
##### redis database
##### mailer daemon
##### sms sender

#### customizations
##### external ports
##### background image
##### photo storage
##### database storage
You could bypass the database services `huautla`, `us-db` or both in favor of using your own host's database servers. `huautla` is very *postgresql* specific, and `userservice` was created for *mysql* and may or may not be agnostic enough to run on other engines. The benefit here is that you get upgrades with your host system, rather than wait for this project to issue updates. Backups could also be scheduled natively, although the existing *backup* and *restore* services would also work fine, if properly configured.

The main changes needed are:
- remove any dependency on either `huautla`, `us-db` or both in [docker-compose.yml](./docker-compose.yml), depending on what you're replacing, and
- update the [template](./env/template) and [secrets](./env/secrets) files with your host's values for hostnames, ports, users, etc

##### runtime customization
There are a few reasons to change the values in `docker-compose.yml`. Mostly around resource contention like port-forwarding. Maybe you want a different name or location for `data/` or `backups/`. Maybe you want to use a different database server or change the current server's username/password. Whatever your reasons, remember that this file was copied, *not* cloned, so if you want to pick up a newer copy, note that you'll have to merge the changes from the repo with the customizations you made locally. Once you get a working config, you should probably make a backup of the compose file.

For developers: you could run a standalone server from the `standalone/` directory in a cloned repo, but that is not perferred. That location is for testing only and the only artifact that should live there is `docker-compose.yml`. If you want to develop this project *and* run an always-live server, it's highly recommended to follow the steps in this README and make a new directory with just a copy of the `yml` file in it. Changes should be made to the copy so as not to taint the repo, unless changes were meant to head upstream.

##### using certs

##### ssl hostname
This section only applies when connecting to port _10080_ (or whichever port you may have changed that to from the original configuration - see [runtime customization](#runtime-customization) above). In short, the `SSL_HOST` value in the [template](./env/template) file should match the hostname of your custom cert installed in [custom certs](#using-certs), above. We don't automatically resolve this name in the webserver because you might have additional port forwarding in front of the application.

##### login look and feel
- _login.html_: the login page supports customization of both `css` and `js` resources, effectively letting you change everythig about the login experience. All customizations are disabled by default. To enable them, uncomment the `volumes:` key in the `us-web` service in [docker-compose.yml](./docker-compose.yml) file, and whichever (or both) of the volume mounts for `css` and/or `js`. Create files with basename `authnz` and the appropriate `.css` or `.js` extension, and they will be fetched with the default _login.html_ page, after the built-in styles and javascript. A common use is sharing the background image between the login and mycology page. Here's a sample `www/css/authnz.css` file:
  ```css
  html {
    background-attachment: fixed;
    background-color: var(--bg-color);
    background-image: linear-gradient(#0a0e18), url(./css/images/background/background.avif);
    background-position: center;
    background-repeat: no-repeat;
    background-size: cover;
    background-blend-mode: hard-light;
  }
  ```
  The colors in this example complement the defaults from the `cffc-web` service, and would need changing if you choose to customize that page as well and want to maintain continuity. There is no way to share between the services without copying/pasting, since the userservice is its own standalone service that can be used in other applications.

- mycology UI: customize this similarly to _login.html_; files go in the same volume mounts under `www/` in the project root directory, but names are `www/css/cffc.css` and `www/js/cffc.js`, instead of `authnz`. This is a *much* larger application than the simple _login.html_ page documented above, but you still only get one file each to change the UI and/or behavior, so extensive changes could result in very large files. As much as possible, we've consolidated the color scheme, buttons and a few random other things with variables in [index.css](https://github.com/jsmit257/cffc-web/blob/rc1/www/rc1/css/index.css), you can override these in `cffc.css` to change some things across the site easily.


#### runtime service details
Here are brief descriptions of the services that need to be running for a healthy application:

##### huautla
Huautla hosts the mycology database on a vanilla `postgresql` image. This is everything needed to track strains, substrates, the lifecycles of individual colonies, and the geneology of new strains. It depends on nothing, and is a dependency of [cffc-api](#cffc-api).

##### us-authn
Authn is responsible for maintaining active logins and password resets. It's a simple `redis` instance, primarily because of it's native support for expiring records. It depends on nothing, and is only depended on by [us-srv](#us-srv).

##### us-db
Userservice database is a vanilla `percona` (mysql) image. It currently only holds users, contacts and addresses for the purpose of user authentication, although it is expected to expand into the realm of authorizations at some later date. It depends on nothing and is only depended on by [us-srv](#us-srv).

##### us-maild
The mailer daemon is a proxy smpt server used when performing password resets that are initiated with an email record that matches the email on record with [us-authn](#us-authn). It has no service dependencies and is a dependency of [us-srv](#us-srv).

##### us-srv
Provides the API for authentication, and eventually authorization. It's nothing more than a simple `alpine` linux image to support the `userservice` binary. It serves data on port 80, with no external forwarding. Both [us-web](#us-web) and [cffc-api](#cffc-api) depend on this service, which depends on [us-authn](#us-authn), [us-db](#us-db) and [us-maild](#us-maild).

##### us-web
The userservice web server is able to run as a truly standalone unit, but in this project it is only accessible through a [cffc-web](#cffc-web) proxy, under the path `/authnz/` It is also the only way to reach [us-srv](#us-srv) APIs. If it were standalone, it would need to forward port 80, and should really have SSL support added. The web has a declared, soft dependency on [us-srv](#us-srv), but it would start without it.

##### cffc-api
Serves the API for managing the [huautla](#huautla) database. When the environment contains non-empty `CFFC_AUTHN_HOST`, and non-zero `CFFC_AUTHN_PORT` variables, it checks every inbound request for an authentication cookie, and returns http *Forbidden* (403) with a `Location:` header pointing to the login page. Therefore, it depebds on both [huautla](#huautla) and [us-srv](#us-srv) (directly, not via the userservice web server).

##### cffc-web
The single client-facing service hosts the static assets that compose the mycology user interface on an `nginx` image. Requests for [mycology APIs](#cffc-api) are proxied through here, and all [authentication](#us-srv) requests, including static assets for [login](#us-web) are sent here. The userservice web server handles proxying userservice API endpoints. Nothing depends on `cffc-web` and it depends, directly or indirectly, on everything in the stack.

The other services in [docker-compose.yml](./docker-compose.yml) are concerned with initialization, backup and restore and should be pretty well described above.

See [further reading](#further-reading) for more detailed descriptions of the projects that provide these services.

#### Troubleshooting
Here are some common problems that yuo may encounter, how to identify them, and suggestions for how to fix them. We limit this discussion to what can be easily fixed. Almost no piece of software is ever bug-free, but these services only get tagged for release after a reasonable amount of testing, so what we're looking for here is problems with how the services work - or rather don't work - together.

##### tools
- We've already mentioned [getting status](#status) above: this is your first line of defense when finding the source of errors. If you can find a service that's clearly misbehaving, it helps to narrow the search.

- All the services emit logs, the custom services like `cffc-api` and `us-srv` have much more verbose logging. While it's possible to see all services logs together:
  ```sh
  $ docker-compose logs
  ```
  the output can be hard to sift through. But if [status](#status) shows a service is flapping (restarting), or missing, you can zero in on that service's logs by name:
  ```sh
  $ docker-compose logs --no-log-prefix <service-name> [| jq .]
  ```
  where `<service-name>` is one of the services listed [above](#runtime-service-details). The `--no-log-prefix` removes the service name from each line of output (useless because we've already identified the service in the command), and allows to pipe output through a filter like `jq`, if you have that installed, to see the *json* output in a more structured way. You can include other filters in the commandline to zero in on e.g. errors, like so:
  ```sh
  $ docker-compose logs --no-log-prefix <service-name> | grep '"error":' | jq .
  ```
  Assuming you have `jq` available, the output is rendered like this:
  ```json
  {
    "app": "serve-mysql",
    "cid": "ded42e50-af67-44bd-8d0e-37eb614d4496",
    "error": "missing auth token",
    "level": "info",
    "method": "GET",
    "msg": "sending status code and messages",
    "remote": "172.18.0.1:54466",
    "status-code": 307,
    "time": "2025-05-10T02:41:10Z",
    "url": "/valid"
  }
  ```
  Without `jq`, the you get the same information, but all rendered in a single line:
  ```json
  {"app":"serve-mysql","cid":"ded42e50-af67-44bd-8d0e-37eb614d4496","error":"missing auth token","level":"info","method":"GET","msg":"sending status code and messages","remote":"172.18.0.1:54466","status-code":307,"time":"2025-05-10T02:41:10Z","url":"/valid"}
  ```

- web-browser console output may contain additional information about a failed request, than what's shown in any dialog boxes that may pop up when an error occurs on the page. The information in dialogs and/or the console can be useful for further filtering the log results to one specific error, or a group of messages related to your error. In the examples above, there is a `"cid":` field, or *Correlation ID*, which is returned as a header in the response to every request sent to the server. The `cid` is generated when the request is recieved, and attached to every log entry generated for the duration of the request, so if you get the `cid` from the header, a command like this will show the start and end, and any ancillary messages in between, for every function called in service of the request: 
  ```sh
  $ docker-compose logs --no-log-prefix <service-name> | grep '"cid":"<cid-from-header>"' | jq .
  ```
  Of course, replace `<cid-from-header>` with the actual `cid` from a failed request's header. It's not uncommon to have 6 or more messages per request.

##### port issues
Another common cause of errors is port contention. We chose random, hopefully sensible, ports for the HTTP Server. If these port-mappings conflict with another service already running on the host, then [the main web server](#cffc-web) will fail to start. To fix this, just change the offending port in `docker-compose.yml` to one that isn't being used.

#### Further Reading
- Huautla: [readme] and [project]
- Center for Fungus Control: [readme], [API reference] and [project]
- Userservice: [readme] and [project]
- Web server: [readme] and [project]
