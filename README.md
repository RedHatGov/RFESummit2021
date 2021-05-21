# RHEL for Edge Demo
This presents a demonstration of RHEL for Edge that includes:
* Rootless and serverless container application
* Automatic failover from one guest node to another
* Automatic restart of container application
* Automatic update to container application
* Atomic upgrade of the underlying operating system with rollback on failure

## Pre-demo setup 
Start with a minimal install of RHEL 8.4+ on baremetal or on a VM
that permits nested virtualization. Make sure this repository is
on your RHEL 8 host using either `git clone` or secure copy (`scp`).
Download the [RHEL 8.4 Boot ISO](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.4/x86_64/product-software)
in the same directory as this repository.

During RHEL installation, configure a regular user with `sudo`
privileges on the host. These instructions assume that the `RFESummit2021`
git repository is cloned or copied to your user's home directory
on the RHEL host.

You'll need to customize the settings in the `demo.conf` script to
include your Red Hat Subscription Manager (RHSM) credentials to
login to the [customer support portal](https://access.redhat.com)
to pull updated content. Also, review the settings for the amount
of memory (in MiB), number of virtual cpus, disk size, and the
virtual IP (VIP) address for keepalived for each of the two edge
guests that you'll be launching using KVM.  Please make sure the
path to the [RHEL 8.4 boot ISO](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.4/x86_64/product-software)
is correct.

The first setup script will configure a network bridge so that the
two edge guests will appear on the same LAN as the RHEL host. The
VIP defaults to address .100 on the same subnet. Adjust these
settings for your environment.

Login to the RHEL host using `ssh` and then run the first script
to update the system and create the network bridge for the virtual
guests.

    cd ~/RFESummit2021
    sudo ./01-setup-rhel8.sh
    reboot

After the system reboots, simply run the remaining scripts in order:

    sudo ./02-config-image-builder.sh
    sudo ./03-config-registry.sh
    ./04-build-containers.sh
    sudo ./05-mkksiso.sh

The above scripts do the following:
* install and enable the web console and image builder
* enable a local insecure registry on the host
* build two versions of a container app and push both to the local registry
* create custom boot ISO with an embedded kickstart file and kernel boot parameters

NB: If the `05-mkksiso.sh` script emits errors, simply rerun it.

Once you've run the above scripts successfully, setup is complete.

## Demo
### Compose the operating system image using composer-cli
You'll need to have four terminal windows ready, arranged in a 2x2
square. Make sure each terminal window is connected to the RHEL
host with the user account you configured earlier.

We'll start by examining the blueprint file that is a template for
the OSTree operating system images we'll be creating.

    cd ~/RFESummit2021
    less RFE.toml

The blueprint file contains metadata for the OSTree image, desired
packages to be installed, firewall customizations, and any users
we would like to add.

We'll continue by pushing the blueprint to the image builder server
on the host so we can begin to compose our image. You can also list
all the blueprints currently on the image builder server.

    composer-cli blueprints push RFE.toml
    composer-cli blueprints list

Now, let's compose our first image using the command line tooling.
There are multiple image types that can be composed. Use the following
command to list them:

    composer-cli compose types

We'll be building a `rhel-edge-commit` image type which is an OSTree
image packaged as a tarball. Start a compose using the following
command:

    composer-cli compose start RFE rhel-edge-commit

The image will take between five and ten minutes to build. You can
check the status using:

    watch composer-cli compose status

which refreshes every two seconds.

When the image is fully built, the status will change from `RUNNING`
to `FINISHED`. Use `CTRL-C` to stop the `watch` command.

Next, we'll create directories to hold our composed images so we
can run a simple web server to provide the content to the edge
guests for installs and upgrades.

    mkdir ~/{version-1,version-2}

Download the first operating system image using:

    cd ~/version-1
    composer-cli compose image <TAB>

The `TAB` key can help complete the command line with the UUID of
the image. You should download a image that's just over 600 MiB in
size. Expand that image and take a look at the metadata using:

    tar xf *.tar
    jq '."ostree-commit"' compose.json

The `jq` command formats the JSON file to be more human-readable.
There are several values of note here including the `ostree-commit`
which uniquely identifies the compose and the `ref` which identifies
the branch for the image content. You can use the `ref` to list all
the packages in the image:

    rpm-ostree db list rhel/8/x86_64/edge --repo=repo | less

We'll be adding the package `strace` in the next version of our
compose, but you can confirm that the package is not included in
this compose using:

    rpm-ostree db list rhel/8/x86_64/edge --repo=repo | grep strace

We need to copy the `ostree-commit` value to the clipboard so we
can designate this compose as the parent to our next build. If you
need to view the `compose.json` file again to get this value, simply
type:

    jq '."ostree-commit"' compose.json

### Compose an operating system image upgrade using the web console
Create the second image using the web console. Browse to
`https://YOUR-HOST-NAME:9090` and log in as the user that was created
during installation of your RHEL host. Once you're logged in, select
"Image Builder" in the navigation bar in the left hand side.

![Image Builder](/images/image-builder.png)

Click the link to the right of the `RFE` blueprint labeled `Edit
Packages`. Under "Available Components", type `strace` in the box
with text "Filter By Name..." and then press ENTER.

![Filter Packages](/images/filter-packages.png)

Click the "+" to the right of the `strace` package to add it to the
blueprint components on the right hand side. Select the "Commit"
button to update the version number and commit this change to the
blueprint.

![Commit Change](/images/pre-commit.png)

You'll be asked to confirm the commit so just select "Commit" again.
Next, select "Create Image" to kickoff a build of the image. In the
dialog, select `RHEL for Edge Commit (.tar)` for the Type field and
paste the `ostree-commit` value you copied into the clipboard into
the Parent commit field. Select the "Create" button to kick off the
image build.

![Create Image](/images/create-image.png)

The build will take between five and ten minutes to complete. Once
it's finished, list the image UUIDs on the command line using:

    composer-cli compose status

The output will look something like this:

    149b153b-82a8-4adb-8f36-27481ac2d0f2 FINISHED Sun Apr 25 14:17:32 2020 RFE
           0.0.1 rhel-edge-commit
    12a775e4-1300-428f-a62c-505042948616 FINISHED Sun Apr 25 14:54:19 2020 RFE
           0.0.2 rhel-edge-commit 2147483648

We want to work with the UUID matching the `0.0.2` version of the
image build. For the `composer-cli compose image` command below,
use command completion by just pressing the TAB key and selecting
the UUID matching version `0.0.2` as shown above.

    cd ~/version-2
    composer-cli compose image IMAGE_UUID

You should download a image that's just over 600 MiB in size. Expand
that image and take a look at the metadata using:

    tar xf *.tar
    jq '.' compose.json

Once again, use the `ref` to list all the packages in the image:

    rpm-ostree db list rhel/8/x86_64/edge --repo=repo | less

You can confirm that the `strace` package is included in this compose
using:

    rpm-ostree db list rhel/8/x86_64/edge --repo=repo | grep strace

### Install the edge guests
Now that the content is ready, let's use the simple golang web
server included in this git repository to serve the image files to
the edge guests during installation and upgrade while also tracking
the number of files requested and their cumulative size. In a
separate terminal window, run the web server in the same directory
as our first image compose:

    cd ~/version-1
    go run ../RFESummit2021/main.go

In each of the remaining terminal windows, we'll install the edge
virtual guests. In the first guest terminal window, type the commands:

    cd ~/RFESummit2021
    sudo ./install-edge-guest.sh primary && sudo virsh console edge-device-1

In the second guest terminal window, type the commands:

    cd ~/RFESummit2021
    sudo ./install-edge-guest.sh backup && sudo virsh console edge-device-2

When the installations are complete, the scripts will exit. You can
examine the web server output to see the total number of files
downloaded to the virtual guests as well as the cumulative size.
Since each guest is downloading the same files, you can divide the
numbers in half to get a rough estimate of how many files and how
large the image is. This should be around 28,000 files with a
cumulative size just over 600 MiB.

### Run the edge guests
Restart each guest in its corresponding terminal window. For the
first guest, use the command:

    sudo virsh start edge-device-1 && sudo virsh console edge-device-1

and for the second guest, use the command:

    sudo virsh start edge-device-2 && sudo virsh console edge-device-2

Each guest is using keepalived and the virtual router redundancy
protocol (VRRP) to assign the virtual IP address to the primary
instance unless it fails. Upon failure, the backup instance assumes
the virtual IP address until the primary instance comes back on
line. The virtual IP address was configured earlier in the `demo.conf`
file. Only one guest at a time will respond to this address.

### Rootless and serverless container application
Each guest is configured to listen on port 8080 at the virtual IP
address for a web request. When an http request is received, systemd
socket activation will launch a proxy service which will in turn
start the container web application to service the request. The
socket listener and both services are running rootless under user
`core`. Neither guest is running any container applications at start
up. The container web application is started when the first request
is received and only one guest will respond since they are electing
an owner for the virtual IP address.

Let's watch this in action. We need to monitor which guest is running
a container application. Log in to each guest using username `core`
with password `edge` and then run the following command on each
guest:

    watch 'clear; podman container list'

In the separate terminal where you built the images, use curl to
send a request to the virtual IP address. This address is defined
in `demo.conf` as the local LAN subnet with `.100`. In the example
below, my local LAN subnet is `192.168.1.0/24` so my virtual IP
address is `192.168.1.100`.

    curl http://192.168.1.100:8080

The primary edge guest, if active, will receive the request on port
8080. If the primary edge guest is not running, the backup edge
guest will respond. Systemd, listening to that port via its socket
activation feature, will launch a proxy service that in turn will
launch podman as a service to run our container web application.
Podman will download the container image from the registry, if it
doesn't already have the container image cached, and launch the
containerized process. That process will then respond to the http
request.

Since both the primary and backup edge guests are running, the
primary guest will launch the container web application in response
to the curl request. We can test the failover feature of keepalived
by terminating the primary guest.  In the terminal window where you
just ran `curl`, type the following command:

    sudo virsh destroy edge-device-1

where `edge-device-1` corresponds to the primary guest and
`edge-device-2` corresponds to the backup guest. Adjust accordingly
for your system.

Once the primary guest is terminated, resend the web request using:

    curl http://192.168.1.100:8080

Make sure to substitute your virtual IP address in the above command.
You'll see the container web application start on the backup edge
guest using the same mechanism as before. This demonstrates that
the virtual IP address was taken by the backup edge guest after the
primary guest was terminated. The web application was then launched
on the backup edge guest to respond to the web request.

### Auto restart of the container web application
Press the key combination `CTRL-C` on the edge guest to terminate
the `watch` command.

The systemd configuration for our container web service has the
policy `Restart=on-failure`. If the program should unexpectedly
fail, systemd will restart it. However, if the program normally
exits, it will not be restarted. The policy can also be modified
to cover many use cases as we'll see in a minute. Let's go ahead
and trigger a restart of our container web application.
Type the following command on the edge guest:

    pkill -9 httpd

This command sends a KILL signal to the httpd processes inside the
container, immediately terminating them. Since the restart policy
is `on-failure`, systemd will relaunch the container web application.
While that's happening, we can discuss the various restart policies
that are available.

The table below lists how the various policies affect a restart.
The left-most column lists the various causes for why the systemd
managed service exited. The top row lists the various restart
policies. And the `X`'s indicate whether a restart will occur for
each combination of exit reason and policy. A full discussion of
the `Restart=` option in the systemd service unit file is available
via the command `man systemd-unit` on the host system (the guest
has no man pages installed to reduce space).

 Restart settings/Exit causes | no | always | on-success | on-failure | on-abnormal | on-abort | on-watchdog 
------------------------------|----|--------|------------|------------|-------------|----------|-------------
 Clean exit code or signal    |    |   X    |     X      |            |             |          |             
 Unclean exit code            |    |   X    |            |     X      |             |          |             
 Unclean signal               |    |   X    |            |     X      |     X       |    X     |             
 Timeout                      |    |   X    |            |     X      |     X       |          |             
 Watchdog                     |    |   X    |            |     X      |     X       |          |     X       

Once again, please confirm that the container application has fully
started. In the same terminal, type the following commands to see
if the container web application is fully active.

    systemctl --user status container-httpd.service
    watch 'clear; podman container list'

### Auto update of the container web application
Let's update the container web application image from version 1 to
version 2. In the same terminal window where you've been running
`curl`, type the following commands:

    HOSTIP=192.168.1.200
    podman pull --all-tags ${HOSTIP}:5000/httpd
    podman tag  ${HOSTIP}:5000/httpd:v2 ${HOSTIP}:5000/httpd:prod
    podman push ${HOSTIP}:5000/httpd:prod

Make sure the value of HOSTIP matches the IP address of your RHEL
host. You can determine this using:

    ip route get 8.8.8.8

and then look at the `src` field.

The edge guest is running a timer that periodically triggers a
systemd service to run `podman auto-update`. This command checks
if there's a different container image than what's currently running
on the edge guest. If so, the newer image is pulled to the guest
and the container web application is restarted. The above commands
force this to occur. You should see output in the edge guest
indicating that the container web application was restarted.

Resend the web request in the same terminal window as before using:

    curl http://192.168.1.100:8080

Make sure to substitute your virtual IP address in the above command.
You'll see that the response from the container web application has
changed.

### Atomic operating system upgrade with rollback on failure
RHEL for Edge simplifies operating system upgrades and automates
rollbacks when upgrades fail.  The greenboot facility enables
applications to define the conditions necessary for a successful
upgrade.

On the edge guest, run the following command to view the greenboot
directory structure:

    find /etc/greenboot

Greenboot is implemented as simple shell scripts that return pass/fail
results in a prescribed directory structure. The directory structure
is shown below:

    /etc/greenboot
    +-- check
    |   +-- required.d  /* these scripts MUST succeed */
    |   +-- wanted.d    /* these scripts SHOULD succeed */
    +-- green.d         /* scripts run after success */
    +-- red.d           /* scripts run after failure */

All scripts in `required.d` must return a successful result for the
upgrade to occur. If there's a failure, the upgrade will be rolled
back. Our edge guest has a simple shell script mechanism to force
a rollback to occur.

Scripts within the `wanted.d` directory may succeed, but they won't
trigger a rollback of an upgrade if they fail. The `green.d` directory
contains scripts that should run as part of a successful upgrade
and scripts in the `red.d` directory will run if there's a rollback.

The simple shell script at
`/etc/greenboot/check/required.d/01_check_upgrade.sh` will fail if
the files `orig.txt` and `current.txt` differ. These files hold the
OSTree commit identifier after initial boot and the OSTree commit
identifier for the current boot. On the edge guest, run the following commands
to review the current OSTree commit identifier
and the contents of those files:

    rpm-ostree status -v
    cat /etc/greenboot/orig.txt /etc/greenboot/current.txt

The text files hold the same OSTree commit identifier that's currently
active. We'll use this simple mechanism, for demonstration purposes,
to control if an upgrade succeeds or rolls back. By default, an
attempted upgrade will fail since the file `orig.txt` will not match
the new OSTree commit identifier in `current.txt`.

On the edge guest, examine the shell script that triggers a rollback when these files are different:

    less /etc/greenboot/check/required.d/01_check_upgrade.sh

The file `orig.txt` is only written if it doesn't already exist.
This enables the guest to remember its original OSTree commit
identifier between upgrade attempts. You can allow an upgrade to
happen simply by deleting this file prior to starting the upgrade.
Again, this is an artifice to enable demonstrating the rollback
process. Production systems should implement checks meaningful to
the operating system and application workloads.

To deliver updated OSTree content, we need to have a running web
server. We'll use a simple web server again on the host to
serve the OSTree content to the edge guest. Type the following commands
in the host terminal to start the web server in the same directory
as the updated OSTree content:

    cd ~/version-2
    go run ../demo-rfe/main.go

The edge guest has two systemd services, triggered by configurable
timers, to stage new OSTree images and force a reboot to trigger
an upgrade. The `rpm-ostreed-automatic` service downloads and stages
OSTree image content. The timer for this service is triggered once
an hour per the corresponding timer configuration. Systemd timers
are incredibly flexible and can be configured for virtually any
schedule. The `applyupdate` service triggers a reboot when there
is staged OSTree image content. The timer for this service is
triggered once per minute to automate upgrades for this demonstration.
Again, nearly any schedule can be used for this timer.

Type the following command in a guest terminal to view when these
timers are set to be triggered.

    systemctl list-timers --no-pager

The output will resemble the following:

    Fri 2021-04-02 18:11:32 UTC  56s left      Fri 2021-04-02 18:10:32 UTC  3s ago applyupdate.timer
    Fri 2021-04-02 18:59:11 UTC  48min left    n/a                          n/a    rpm-ostreed-automatic.timer

Rather than wait up to an hour for the `rpm-ostreed-automatic`
service to be triggered via its timer, we'll instead force an upgrade
by directly running the service. Type the following command in the
guest terminal window:

    sudo systemctl start rpm-ostreed-automatic

This will start the process on the edge guest device to pull new
content from the host. In the host terminal, you'll see the various
files being downloaded to the the guest. You should also notice
that the number of files and their accumulative size is much smaller
than the initial installation. This is because rpm-ostree only
downloads the deltas between the current OSTree image and the new
OSTree image which can save significant bandwidth and time when
operating in environments with limited connectivity.

After the content is downloaded and staged, the `applyupdate` timer
will expire within a minute and start the `applyupdate` service
that will trigger a reboot.

Since the `/etc/greenboot/orig.txt` file contains the OSTree commit
identifier from the initial installation, any successive upgrade
will fail. The system will attempt to upgrade the operating system
three times before rolling back to the prior version. With each
boot attempt, you should see the following appear in the guest
terminal:

      Red Hat Enterprise Linux 8.4 (Ootpa) (ostree:0)
      Red Hat Enterprise Linux 8.4 (Ootpa) (ostree:1)

The `ostree:0` image will be highlighted on the first three boot
attempts since `ostree:0` designates the most recent operating
system content and `ostree:1` represents the previous operating
system content. After the third failed attempt, the guest terminal
will then highlight `ostree:1` which indicates that a roll back is
occurring to the prior image. This entire process will take a few
minutes to complete.

When the rollback is completed, the login prompt will appear for
the edge guest. Login using username `core` and password `edge`.

When we built the `0.0.2` version of our operating system OSTree
image, we added the `strace` utility. You can confirm that utility
is missing in the current active operating system image by typing
the following command in the guest terminal:

    which strace

The output should resemble the following:

    /usr/bin/which: no strace in (/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin)

Type the following command to confirm that the upgrade did not occur
by looking at the current commit identifiers of the local operating
system content:

    rpm-ostree status -v

The output from that command will resemble the following:

    State: idle
    AutomaticUpdates: stage; rpm-ostreed-automatic.timer: no runs since boot
    Deployments:
    * ostree://edge:rhel/8/x86_64/edge
                     Timestamp: 2021-03-31T00:11:31Z
                        Commit: 98a1d03316797162d4b3a1fad22c36be049c46b42605307a0553e35c909c6a6d
                        Staged: no
                     StateRoot: rhel
    
      ostree://edge:rhel/8/x86_64/edge
                     Timestamp: 2021-03-31T00:38:20Z
                        Commit: 662c26c39800e4fa97430fecdab6d25bd704c8d9228555e082217b16d5697f02
                     StateRoot: rhel

The active image is preceded with an `*` and you can tell its the
prior image by looking at the timestamps for both. The commit
identifier for the active image also matches the commit identifiers
in both the `/etc/greenboot/orig.txt` and `/etc/greenboot/current.txt`
files. Type the following commands to exxamine those files.

    cat /etc/greenboot/orig.txt /etc/greenboot/current.txt

Now, let's enable the OSTree upgrade to occur by deleting the
`/etc/greenboot/orig.txt` file so that our greenboot script at
`/etc/greenboot/check/required.d/01-check_upgrade.sh` returns
success, allowing the upgrade process to move forward. In the guest
terminal, type the following command:

    sudo rm -f orig.txt

We need to stage new OSTree content for the upgrade, so we'll re-run
the `rpm-ostreed-automatic` service to download and stage the content
from the web server on the host. Please type the following commands
on the edge guest to review the timers and start the atomic upgrade
process:

    systemctl list-timers --no-pager
    sudo systemctl start rpm-ostreed-automatic

You'll observe in the host terminal that only five files are requested
of inconsequential size. The guest already has the content needed
so it's merely requesting metadata to determine if this is the most
recent OSTree content. Once the content is staged, the `applyupdate`
service timer will expire within a minute and the `applyupdate`
service will trigger a reboot.

The upgrade will succeed this time. The boot screen with the OSTree
image list will appear only once. This will take a minute or two
to finish. Login again as user `core` with password `edge`. Confirm
that the upgrade was successful by typing the following command on
the edge guest to show that the `strace` command is available:

    which strace

The output should resemble:

    /usr/bin/strace

You can also examine the OSTree image list to see that the newest
OSTree image content is active.

    rpm-ostree status -v

The output from that command will resemble:

    State: idle
    AutomaticUpdates: stage; rpm-ostreed-automatic.timer: no runs since boot
    Deployments:
    * ostree://edge:rhel/8/x86_64/edge
                     Timestamp: 2021-03-31T00:38:20Z
                        Commit: 662c26c39800e4fa97430fecdab6d25bd704c8d9228555e082217b16d5697f02
                        Staged: no
                     StateRoot: rhel
    
      ostree://edge:rhel/8/x86_64/edge
                     Timestamp: 2021-03-31T00:11:31Z
                        Commit: 98a1d03316797162d4b3a1fad22c36be049c46b42605307a0553e35c909c6a6d
                     StateRoot: rhel

The `orig.txt` and `current.txt` files will also contain the OSTree
commit identifier for the latest OSTree image. Type the following
commands on the edge guest:

    cat /etc/greenboot/orig.txt /etc/greenboot/current.txt

To terminate the emulated edge device, simply type the following
command in the guest terminal window:

    sudo poweroff

This completes the RHEL for Edge demo.

