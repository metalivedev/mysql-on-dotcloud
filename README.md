#Custom MySQL Installation for dotCloud Platform -- ALPHA QUALITY

You should probably use the standard *mysql* service from dotCloud --
this one is definitely experimental. But if for some reason you need
to run the latest MySQL and don't care about high availability or
failover, then this is one place to start.

##Step 1: Get the MySQL Binary Tarball from Oracle

You can download MySQL generic binary from
http://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.12-linux-glibc2.5-x86_64.tar.gz/from/http://cdn.mysql.com/

##Step 2: (for sanity) Repack the Tarball

The tarball is almost 400M and includes a bunch of tests and binaries you
probably don't need. The huge tarball will slow down your deployments
as it gets copied from the uploader to the build machine, and then
from there to your service. We can shrink it down to about 79M by
removing the debug binaries and tests. Tar that back up and put it in
the ``testbed`` directory as *something*.tar.gz

##Step 3: dotcloud create and push

Do the normal thing for dotCloud. You've got the CLI, right?

###What happens during the installation?

Once your files are uploaded to the internal dotCloud Uploader
service, they get labeled with the revision and then copied to the
dotCloud Builder (also internal).

The Builder checks the ``dotcloud.yml`` file for what it should
build. It sees that it needs some system packages, so those get
installed first.

In this case, the next step is to build a custom service, which means
that it needs to find a script to tell it how to build the image. In
this case, the script is ``builder``.

Since the tarball already includes a binary, we'll just unpack the
tarball into a specific directory. We won't be running mySQL as root,
so we'll need to do some special configuration.

In particular, whenever we run any mySQL executable (for admin, for
the daemon, for the CLI), we will need to specify the
``--defaults-file`` so that it does not try to use the system defaults
at all.

Once the ``builder`` has run, the dotCloud Builder will take a tar'd
snapshot of the build products and store it. The dotCloud Deployer
then takes that snapshot, spins up as many containers as requested by
your `dotcloud scale` and copies the build tarball to each.

Then the Deployer runs the ``postinstall`` script in that runtime
environment. Here it is safe to create the directories for databases
and logs. We choose to do this under ~/data because the dotCloud
platform treats ~/data in a special way: it gets moved from the old
container to the new container at each push, so it is
persistent. That's what we want in a database! 

Next ``postinstall`` reads the internal PORT number from the
environment and update the mySQL configuration file ~/.my.cnf so that
we bind to the right place. There is a different port number where
clients should connect.

Once the directory is in place and the port number is in the
configuration, we can initialize the database:

    scripts/mysql_install_db --user=dotcloud --defaults-file=~/.my.cnf 

Finally supervisord starts the database by running the script we
specified in the *process* section of ``dotcloud.yml``: ``starttest``.

##Step 4: Scale RAM

The default size of a *custom* type is 128M, but it appears that this
mySQL installation requires at least 512M, so you'll need to
``dotcloud scale testbed:memory=512M``

##Security:

PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:

    /home/dotcloud/mysql/install/bin/mysqladmin --defaults-file=~/.my.cnf -u root password 'new-password'
    /home/dotcloud/mysql/install/bin/mysqladmin --defaults-file=~/.my.cnf -u root -h *YOURAPPNAME*-default-testbed-0 password 'new-password'

