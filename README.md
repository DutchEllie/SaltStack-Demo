# Salt tutorial

<!-- 
So, the old tutorial was shit! Absolute dogshit, nothing can be done to fix it.
[REDACTED NAME] and I have decided that we rewrite the entire thing so it can follow a simple example of installing Gitea and a database and configuring it.
The reason for this is because we decided that we would do something like this for the video, but the scriptmakers for the video didn't do that.
This would result in both a horrible video and a horrible tutorial, that's why we changed it a bit.

It's currently Sunday as I write this, and I am extremely sad 😢 to be writing this on a Sunday.
-->

<!--

IMPORTANT!!!!
Style guide!
READ THIS WHOLE COMMENT IF YOU WANT TO WRITE LIKE A SANE HUMAN BEING AND NOT A MONKEY!!!

Here you learn how to write this document!

So headings in markdown are done using #.
One # means heading size 1 (used at the top of the document only)
Two ## means heading size 2 (Used for sections)
three ### means heading size 3 (used for paragraphs, don't use too much)
This goes on for a bit

ALWAYS, I FUCKING REPEAT ALWAYS LEAVE EMPTY LINES SURROUNDING THE HEADINGS LIKE THIS:

```
last text of a paragraph

## new section

new text
```

ALSO I WILL FUCKING MURDER YOU WITH MY BARE FUCKING HANDS IF YOU DON'T MAKE A NEW LINE FOR EVERY SINGLE MOTHERFUCKING SENTENCE YOU MAKE!!!
Do you have any idea how motherfucking annoying it is to edit text if you don't make a newline for every sentence, like I have been doing this entire time!???!??!
Markdown on GitHub will render all sentences with a newline after then like normal text anyway!

To add a whitespace, just leave an extra line between your text like I have been doing this entire time!
You can also add two spaces after a line, that will not add a whiteline but just a newline.

To add code inline (so in your sentence) surround the text with backticks like `this`.
Use inline code whenever you explain a command or tell the user to run a command.
DO NOT I REPEAT DO NOT I SWEAR TO FUCKING GOD make a code block for every command.
That is a waste of space and a crime to all that's good.
I shall personally make sure you die if you do this

Full code blocks are made using three backticks, surrounding the block of text.
THEY ARE ALWAYS PRECEDED AND FOLLOWED BY A WHITELINE!!!!
You need to add the code language after the first three backticks like this: ```yaml to add coloring to the rendered webpage.
Full code blocks are used for when you need to show output of a command to the user like this:

```bash
user@machine $ ls
Documents Downloads Pictures Videos
```

Or when showing configuration

```yaml
some_config_line:
  spaced_line:
    - waow
```


WHAT TO WRITE!!?!?!
Write text and keep a story.
The storyline is important for the reader to follow what's going on.
Have you attempted to read the fucking salt documentation???
That fucking mess of a document is what happens if you don't fucking keep a fucking storyline in your fucking tutorial.
They throw everything at you all at once entirely in theory and COMPLETELY disregard actually teaching the reader something!
-->

This tutorial explains will explain the software tool Salt, sometimes called SaltStack.
Salt is an open source configuration management tool used by system administrators to configure and set up servers.
It is similar to tools such as Ansible, but works a little differently.

Before we get into this tutorial, we must first cover the scope of this tutorial.
This tutorial will explain Salt with a practical approach to things.
Rather than explaining all of the theory and showing little practical use, ultimately confusing the reader, we have decided to do things differently.
This tutorial is therefore a practical application of Salt as you could encounter in the real world.
While still simplified, we believe that through this approach it's much easier to learn Salt rather than working your way through the theory before ever running a single command.

## The Practical Setup

Our practical application is installing the open source software Gitea <!-- Insert reference to Gitea here --> on a server, along with a database on a different one.
The deployment is done using AWS EC2 instances, because this makes it easy to set up and realistic.

Create a VPC, with CIDR range `172.18.0.0/16`, two availability zones (one is fine too) with both public and private subnets.
Also add a NAT gateway in there, so that the instances in the private subnets can actually reach the internet.
We will deploy four EC2 instances with the instance type `t2.micro`, but it should work with `t3.micro` and possibly even a `t2/t3.nano`.
Deploy three of these instances (the minions) with the `Amazon Linux 2`.
Put two of these minions (gitea and database) in private subnets without a public IP.
Put the third minion (reverse proxy) in a public subnet with a public IP.
The fourth instance will be the master.
Deploy this one in a public subnet (you need to access it via SSH) with a public IP and use the `Debian 11` AMI. (We ran into some issues with Amazon Linux when we got to the [GitFS section](#gitfs-fileserver-for-the-salt-formula) of the tutorial, so that's why this one uses Debian).



Requirements for the security groups:

* Master:
  * Public can access port 22 (for ssh)
* Reverse proxy:
  * Public can access port 80
* Gitea:
  * Reverse proxy can reach port 3000
* Database:
  * Gitea can reach port 3306

It is useful to give the master instance an IAM role with the permission to use AWS' [`Instance Connect`](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html).
Otherwise you cannot reach the minions in the private subnets at all.
We will assume you know how to connect to the minions in the private subnets.

With that out of the way, let's begin.

## Salt Architecture

While Salt as a tool is similar in purpose to tools like Ansible, it works a little differently.
Ansible utilizes remote command execution software like SSH or Windows Remote Management (allowing for PowerShell execution), but Salt has a different approach.
Salt works by creating the concepts of a "master" and "minion" machines.
The Salt master is the machine that the system administrator uses to configure the minion machines on.
The minion machines communicate with the master over a secure message bus, built using ZeroMQ <!-- Insert ZeroMQ reference here -->.

In this document, we use the word "system" to refer to all of the minions connected to the Salt master.
This makes it easier to explain, especially when we want to refer to multiple minions at once.

Before we get ahead of ourselves and explain everything here, let's look at a practical example first by installing Salt on our servers.

## Installation

The instances will probably be unable to reach eachother using their hostnames natively.
You can set up the DNS names so it's easy to remember in every minion's `/etc/hosts` file, or just use their IP addresses.

First we will install the Salt master.
To install the Salt master, use the [bootstrap script](https://docs.saltproject.io/salt/install-guide/en/latest/topics/bootstrap.html) from the official installation page of Salt.

Run the command `curl -L https://bootstrap.saltproject.io | sudo sh -s -- -M -N` to immediately download and install the script.
Alternatively, it's possible to download the script and save it to a file using `curl -o bootstrap-salt.sh -L https://bootstrap.saltproject.io`, make it executable using `chmod +x ./bootstrap-salt.sh` and then execute the script using `./bootstrap-salt.sh -M -N`.
The `-M` flag tells the bootstrap script to install the Salt master service and the `-N` flag tells the script to not install the Salt minion service.

Once this has been installed, you are good to go for now!
Let's continue with installing the Salt minions.
SSH to the minion and run the same command as above `curl -L https://bootstrap.saltproject.io | sudo sh -s --` without the `-M` and `-N` flags.
This will install the Salt minion service on the minion.
Do this for all of the minions.

Before continuing, the Salt minions need to be told where they can find the Salt master.
This is done by creating a new file on the minion at `/etc/salt/minion.d/05-master.conf` and adding the following content into it:

```yaml
master: <ip or hostname of master>
```

Do this for all the minions and restart the Salt minion using `sudo systemctl restart salt-minion`.
Once you are done doing that, the setup on the minions is complete!

There is just one more thing we need to do now.
When the minions are told where the master is and restarted they will send their public keys to that location for the master to receive.
The master system now just needs to accept these keys.

To list all of the keys including the unaccepted keys, use `sudo salt-key -L`.
You will see that the minions have sent their keys to the master and they are now under the "Unaccepted Keys".
To accept all of them at once, use `sudo salt-key -A` and confirm your action by typing a `y` and pressing the `return` key.
Now all of the keys are accepted and the master can control these minions!

To test your new powers of controlling three machines at once run `salt '*' test.ping`.
The output of this command would look like this:

```
admin@ec2-saltmaster $ sudo salt '*' test.ping 
reverseproxy:
    True
gitea:
    True
database:
    True
```

You can also install software on these minions, such as the popular text editor `vim`.
Just run `sudo salt '*' pkg.install vim` and watch the magic happen.
All of the minions will consult their package managers and look for the `vim` package and install it directly!
`pkg.install` here is a virtual function implemented for multiple package managers.
An example for `apt` can be found [here](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.aptpkg.html#salt.modules.aptpkg.install).
Since `salt.modules.pkg` is a virtual module, it implements the functions for other package managers in one.
That way, if you managed multiple servers running multiple distributions of Linux, you can use the same command for all of them!
There are many [more of these modules](https://docs.saltproject.io/en/latest/py-modindex.html#cap-m:~:text=m-,salt.modules,-salt.modules.acme) available and documented on the Salt website.

Alright, enough playing around.
Let's get back to business and install some software!

## Salt Grains

So we have seen before how we can install software on every single minion at the same time.
But that isn't very useful unless you want a completely identical fleet of machines all running the exact same software.

So you can target specific minions using their ID, which is just their hostname by default.
Using `sudo salt 'gitea' pkg.install vim` you would only be installing `vim` on the minion with the ID `gitea`.
You can even take this further, by using regular expressions!

However, there is still an issue.
There are inherent differences between systems that cannot be described in the hostname, for example a difference in CPU architecture.
It doesn't really work out well if the same package has two different names on different distributions.
For example, `apache2` on Debian/Ubuntu is called `httpd` on RHEL based systems (using the yum package manager).
Therefore, we need a way to differentiate between different minions based on their characteristics as well.

This is where grains come in!
Grains are pieces of information about the minion system.
Things like: CPU architecture, Linux distribution, package manager, etc.
Every minion has them and they can be queried by the master and used to filter minions.

To view all of the available grains for every minion, use `salt '*' grains.ls`.
Now we get a huge list of minions and the grains they make available.
You may notice that there isn't any data displayed apart from the grain's name (called the key).
To show the data as well, use `salt '*' grains.items` to print absolutely everything.
This is pretty great!

To use this information, we can use the `-G` option filter on these grains like this: `sudo salt -G 'os:Ubuntu' test.ping`.
When running that command, you will probably see every minion react to this anyway if all of your minions are using Ubuntu.
These default grains are useful, but not really if you want to assign minions to a different role.
That is why we can also create custom grains!

To create a custom grain on a minion you can use the command line on the master `sudo salt 'gitea' grains.setval role gitea`.
However, it's recommended to set them up using the `/etc/salt/minion.d/grains.conf` file on the minion itself.
Grains can be any common data structure like string, int, boolean, list or dictionary.
To set them up on the minion, edit the file `/etc/salt/minion.d/grains.conf`.
For this tutorial, we will set up the grains based on the roles we assign the minion.

```yaml
grains:
  role: gitea
```

The above example is for `gitea`.
Make the same file for `database` and `reverseproxy`, but change the value for role to `database` and `reverseproxy` respectively.
After this, restart the Salt minion using `sudo systemctl restart salt-minion` and the grains will be synced with the master.

After doing all of that run `sudo salt '*' grains.item role` to see the role for each minion to verify it worked.
The following is the result you want to see.

```bash
user@machine $ sudo salt '*' grains.item role
gitea:
    ----------
    role:
        gitea
reverseproxy:
    ----------
    role:
        reverseproxy
database:
    ----------
    role:
        database
```

Now that we have set up the roles for these minions, it's time to start using them!
Or that is what I would have said if there wasn't another huge part of Salt.

## Salt States

Previously, we have been altering the minions using the command line tool, however you might see the problem here.
If we need to perform a lot of actions, how do we do that?
We could use the command line tool a lot and just do everything through there.
You can even write down the commands you use in a document (of even script) to execute for later.
There is a better way however.
But for that, we must leave the title of this section a mystery for just a bit longer and talk about Salt functions.

### Setting up your file server

To do anything with states or functions, you first have to configure the master's file server.
To do this, edit the file `/etc/salt/master.d/10-roots.conf` and add the following lines:

```yaml
file_roots:
  base:
    - /srv/salt/base
```

Now restart the `salt-master` service using `systemctl`.
Trust me, we will get to it later, but we need the file server before we can use Salt states and functions.

### Salt Functions

A Salt function is a file describing certain actions to be performed on one or more minions.
This file is generally written in YAML, but Salt supports more renderers and Jinja is a popular alternative.
This tutorial generally only focuses on YAML whenever we can, but will use some Jinja when we need to (we'll get to it I promise).

The best way to explain a Salt function is to show an example.
Make a new file and call it `example.sls` and place it in the directory `/srv/salt/base` (make it if you don't have it).
Put the following text in it.

```yaml
install vim:
  pkg.installed:
    - name: vim
```

This is a very simple example of course, but it serves our purpose quite well.
The structure of this file is simple.

```yaml
install vim:          # This is the ID of the "function" or "state". It has to be unique.
  pkg.installed:      # module.function: Calling the module "pkg" and the function in it "installed".
    - name: vim       # An argument for the function "installed".
                      # Multiple arguments can be called, but in this instance we only need the name.
```

Since this structure is pretty simple, it is also highly extensible.
Making it very powerful.

We can now apply this file to the minions using the command `sudo salt '*' state.apply example`.

Notice the word `state` in this command. This is because Salt functions are the 'building blocks' of Salt states.
The only way to perform a function is to apply it as a state.
Also, Salt functions or states only accept .sls files.
This means that you don't have to specify the .sls extention in the above command.

### Salt states file tree

As mentioned before Salt functions are the 'building blocks' of Salt states.
That means that a Salt state is build up of one or many Salt functions.
These functions are compiled in SLS files (SaLt State files).

Of course a configuration might get a really complex when you have to enter the entire configuration in one file.
This is why Salt state files are configured in a tree file structure.
A tree file structure always starts at the top, so naturally the first file of a state is called `top.sls`.
This is the file that gets called first when you apply a state without specifying a function file.
This is done using the `sudo salt '*' state.apply` command.

Notice how there is no more state specified at the end of the command, as opposed to using purely functions.
This specifies that we will by applying a state, and Salt will automatically look for the `top.sls` file in our file server.

### Targeting specific minions

Of course when you declare a state, you might want to specify to wich minion you want to apply it.
This can be accomplished in two ways.

#### 1. Target when applying the state

From the master, apply the state to minions with the following command:

```bash
salt 'target' state.apply
```

`target`: The target is a selector based on Salt grains, as we have seen before in [Salt grains](../2.%20Salt%20Grains/grains.md).
For all targeting options reference the [official Salt documentation](https://docs.saltproject.io/en/latest/topics/targeting/index.html).

#### 2.  Define minion targets in the top file

In the `top.sls` file you can filter minions using grains.
You can then apply other `.sls` files to specific minions.
Using this you can have one `top.sls` file that you can apply to all minions.
It will then configure the correct minions with the specified `.sls` files.
The setup for this config is specified as follows:

```yaml
base:                     # ID for targeting
  '*':                    # Grains
    - example             # The .sls file to be applied
```

`example` here tells Salt to look for a file called `example.sls` in the same directory as the `top.sls` file.
It is also possible to create a directory called `example` and place a file called `init.sls` in it instead.
The file path would look like this `/srv/salt/example/init.sls`.
If both files exist, `example.sls` takes precedence.

This file is then applied to your minions with the command `salt '*' state.apply`.

This is the method we will use to differentiate between our machines.
Combining both methods is also possible, however this is not necessary for our use case.
Speaking of our use case, lets see how we configure our Gitea, mySQL and NGINX servers with Salt states.

## The last steps before installing software

To deploy Gitea and the required services, we will utilize Docker.
Gitea recommends you use Docker rather than installing it on the system itself.
To do this, we will use the [Docker formula](https://github.com/saltstack-formulas/docker-formula) for Salt.
A [Salt formula](https://docs.saltproject.io/en/latest/topics/development/conventions/formulas.html) is a premade collection of Salt states that are easy to use.
The Docker formula has functions for deploying using Docker Compose or straight containers.
To use the formula, you will need to set up the GitFS fileserver for Salt.
We will also need to set up the Salt Pillar.

### GitFS Fileserver for the Salt Formula

Remember when we put some configuration for `file_roots` in the file `/etc/salt/master.d/10-roots.conf`?
Yeah, we need to add some more configuration there!
We will need to add the Git fileserver backend configuration there as well as the URL to the formula.
This is covered in more detail on the [Salt formula page](https://docs.saltproject.io/en/latest/topics/development/conventions/formulas.html) and the [Salt Git fileserver backend page](https://docs.saltproject.io/en/latest/topics/tutorials/gitfs.html#tutorial-gitfs), but we will do the short version.

Install the package `python3-pygit2` with `apt` on the master.
This is a requirement if you want to be able to use the Git fileserver backend.
Then add the following lines to the file `/etc/salt/master.d/10-roots.conf`:

```yaml
fileserver_backend:
  - roots
  - gitfs             # Add this one, the other ones are already there

gitfs_remotes:
  - https://github.com/saltstack-formulas/docker-formula.git
```

This adds `gitfs` as another fileserver backend and adds the Docker formula to the list of remotes to be pulled with GitFS.
Make sure to restart the `salt-master`.

To verify the formula was added, run `salt gitea cp.list_master`.
This asks the `gitea` minion which files it can get from the master.
This list should now be big and include the files from the Docker formula.

### Salt Pillar

The Salt pillar is the last part of the puzzle we need before we can *finally* start to deploy our applications, like we have been teasing the entire tutorial now.
The Salt pillar is a way for the master to securely communicate data with the minions.
This data can be substituted in directly in the CLI or substituted into your Salt States.
The Docker formula makes heavy use of the Salt pillar, that's why we need it.

To set up the pillar, we need the `/etc/salt/master.d/10-roots.conf` file just one more time, I promise!
Add the following lines to it:

```yaml
pillar_roots:
  base:
    - /srv/salt/pillar/base
```

This sets up the pillar for the environment `base` in the directory: `/srv/salt/pillar/base`.
You can now place files with regular `SLS` syntax in this directory and sync them with the minions.

Okay, we have set up the fileserver for both files and GitFS.
We have added the Docker formula.
We have set up the grains for the minions and we have explained enough already.
Finally, let's install some actual software!

## Installing Gitea

As stated before, we will deploy Gitea on Docker, using the Docker formula for salt.
To use this formula, we just need to include it in our `SLS` files.

Navigate to the `/srv/salt/base` directory where we will place everything.
If you have been diligently following the tutorial step by step, you will have already created a `top.sls` file.
Edit it and replace its contents with the following:

```yaml
base:
  '*':
    - essentials
  'role:gitea':
    - match: grain
    - gitea
```

This makes every minion install the essentials, while the `gitea` minion also installs `gitea/init.sls`.
Create the directory `essentials` and create and edit a new file called `init.sls` inside of it:

```yaml
include:
  - docker

install_docker:
  pkg.installed:
    - name: docker
    - reload_modules: True

install_docker_py:
  pip.installed:
    - name: docker==5.0.3
    - reload_modules: True
```

Let's break this down.
We include the Docker formula in the first line.
This is necessary because it will install the required package repository and packages for Docker, but strangely enough not Docker itself, we need to do that ourselves.
We install Docker in the second part, calling for `pkg.installed` and installing Docker.
In the last part we install the Python Docker SDK which is necessary for Salt to talk to Docker.  
**Note that at the time of writing** we are using version `5.0.3` of the Docker SDK because of [an issue](https://github.com/saltstack/salt/issues/62602) with Salt and the Docker SDK version `6.0.0` that breaks functionality.
When that issue is resolved, this can (probably) safely be updated to the latest version.  
You may also notice the `reload_modules` statement.
This is used for reloading the modules, which is needed to let Salt know that Docker has been installed and it can start using it with the Docker modules for Salt.

Now add the following into a new file `init.sls` in the directory `/srv/salt/base/gitea`:

```yaml
include:
  - docker.compose.ng
```

This `include` statement includes the Docker Compose functions from the Docker formula.
This function imports the configuration for the Docker containers from the Salt pillar, which is why we set it up before.  
That's why, as you may have guessed, we need to add this configuration to the pillar.

Create the files `top.sls` and `gitea.sls` in `/srv/salt/pillar/base` and add the following to each files respectively:

`top.sls`:

```yaml
base:
  'gitea':
    - gitea
```

`gitea.sls`:

```yaml
docker:
  compose:
    ng:
      gitea_container:
        image: gitea/gitea:1.17.1
        container_name: gitea
        env:
          USER_UID: 1000
          USER_GID: 1000
        restart: always
        volumes:
          - /data/gitea:/data
          - /etc/timezone:/etc/timezone:ro
          - /etc/localtime:/etc/localtime:ro
        ports:
          - "3000:3000"
          - "222:22"
```

The `top.sls` file tells Salt that the `gitea.sls` file should be synced with the minion `gitea`.
Before you run `salt '*' state.apply`, be sure to run `salt '*' saltutil.refresh_pillar`, because Salt doesn't do that automatically.  
**Also note** that you may run into an issue with the deployment of the Docker container.
This is because [Salt has a bug](https://github.com/saltstack/salt/issues/54449) that they haven't appeared to have fixed, despite having closed the issue.
Just apply the state again and it may work, it's not entirely stable, but Salt hasn't fixed this issue yet.

You, after having run `salt '*' state.apply`, you will have deployed a Docker container on the `gitea` minion.
By using Docker, you can easily manage multiple deployed applications on servers without them interfering.
We are just using Salt to manage them here.

## Installing MySQL

Damn, we're on a roll now!
Time to deploy the database that Gitea needs to work!
We will choose a MySQL database for this since Gitea provides [instructions](https://docs.gitea.io/en-us/install-with-docker/#mysql-database) for it and supports it well.
Deployment actually works similarly to Gitea, since we are also using Docker to make our life much easier!

In `top.sls`, we can edit it to look like this:

```yaml
base:
  '*':
    - essentials
  'role:gitea':
    - match: grain
    - gitea
  'role:database':
    - match: grain
    - database
```

Now also create the the `init.sls` in directory `/srv/salt/base/database`.
Add the following in there:

```yaml
include:
  - docker.compose.ng
```

With that done, we will edit the pillar for the `database` minion in the same way we did for the `gitea` minion.

`pillar/top.sls`:

```yaml
base:
  'gitea':
    - gitea
  'database':
    - database
```

`pillar/database.sls`:

```yaml
docker:
  compose:
    ng:
      database:
        image: mysql:8
        container_name: gitea_database
        environment:
          MYSQL_ROOT_PASSWORD: gitea
          MYSQL_USER: gitea
          MYSQL_PASSWORD: gitea
          MYSQL_DATABASE: gitea
        restart: always
        volumes:
          - /data/mysql:/var/lib/mysql
        ports:
          - "3306:3306"
```

This is the Docker compose configuration for the database.
It pulls the `mysql:8` image from the Docker Hub and deploys it.
That image takes a few environment variables that we need to set.
The `MYSQL_ROOT_PASSWORD` is mandatory and the others need to be set for Gitea to work.

Let's talk about security a bit, since this password policy isn't exactly that secure.
In a production setup, these should be changed to be something more secure.
However, the database isn't actually accessable from the internet if you have set the VPC from before up right.
The only instance in our setup that can access the 3306 port on the database instance is the Gitea instance, not even the reverse proxy can do so.
If at some point the reverse proxy got hacked, this hacker would still need to hack the Gitea instance before they can access the database.

Back to deployment, run `salt '*' saltutil.refresh_pillar` followed by `salt '*' state.apply`.
The database container will be deployed on the `database` minion.

## Installing Nginx

The reverse proxy will utilize the webserver software Nginx.
This is because it is widely used and very good.
It's also quite easy to set up.

Contrary to what we have done before, we will not use Docker to deploy Nginx.
Whereas before we used Docker because it was the recommended way to deploy Gitea and its database, Nginx can easily be deployed on the machine itself.
It wouldn't really be a very good tutorial either if we only used Docker too.

Similarly to what we have done before, add the reverse proxy minion to the `top.sls` file.

```yaml
base:
  '*':
    - essentials
  'role:gitea':
    - match: grain
    - gitea
  'role:database':
    - match: grain
    - database
  'role:reverseproxy':
    - match: grain
    - reverseproxy
```

And again similarly to what we have done two times before, edit the file `reverseproxy/init.sls`:

```yaml
epel_release:
  cmd.run:
    - name: amazon-linux-extras install epel -y
    - unless: yum repolist | grep epel
    - runas: root

install_nginx:
  pkg.installed:
    - name: nginx
  service.running:
    - name: nginx
    - enable: true
```

We may need to explain this a little bit.
The first part install the `epel` repo for yum.
Since we are on Amazon Linux 2 and not on a regular RHEL or CentOS like distro, we just have to do things a little differently.
We execute `amazon-linux-extras install epel -y`, but only if `yum repolist | grep epel` doesn't return anything.
That last command lists all of the repos with `epel` in the name.
If the returned list is empty, that means that it has not been installed!

The second part installs Nginx and ensures the service has been started.
Pretty regular stuff.

However, before we apply any of this, can you see an issue here?
When Nginx installs, the default Nginx configuration is pulled in and sends the visitor to a default page.
This is not what we want.
We want visitors to be redirected to Gitea!

To do so, we will also have to manage the configuration file.
We can easily do that using the `file` state module from Salt.
Add the following lines to the end of your `reverseproxy/init.sls` file:

```yaml
manage_nginx_conf:
  file.managed:
    - name: /etc/nginx/nginx.conf
    - source: salt://reverseproxy/nginx.conf
  cmd.run:
    - name: systemctl reload nginx
    - runas: root
    - onchanges:
      - file: /etc/nginx/nginx.conf

manage_gitea_conf:
  file.managed:
    - name: /etc/nginx/conf.d/gitea.conf
    - source: salt://reverseproxy/gitea.conf
    - makedirs: True
  cmd.run:
    - name: systemctl reload nginx
    - runas: root
    - onchanges:
      - file: /etc/nginx/conf.d/gitea.conf
```

Here, we use the `file` state module to manage the configuration files `nginx.conf` and `gitea.conf`.

In the first section add `file.managed` and set it to manage the file `/etc/nginx/nginx.conf` on the minion.
It should source this file from the master's fileserver, at `salt://reverseproxy/nginx.conf`.
This location refers to the full path `/srv/salt/base/reverseproxy/nginx.conf` on the master and that's where you have to place the file.
Because this file is a little big to put in here, we have placed the file [alongside this document](./nginx.conf) in the repository!
We also use `cmd.run` here to run `systemctl reload nginx` whenever this file changes on the minion to reload the Nginx configuration.

In the second section we manage [`gitea.conf`](./gitea.conf) on the minion as well.
This file contains our configuration for the reverse proxy to Gitea.
**Before you apply** however, you should change the `server_name` and `proxy_pass` directives in there to reflect your situation!!
And in the same way as before, we reload the Nginx configuration whenever the file changes.

Apply your configuration using `salt '*' state.apply` and watch the magic happen!
Nginx will be installed and the configuration files will be placed in the correct locations on the minion.
You can visit your Gitea instance at the DNS name of your EC2 instance (the same hostname as you entered in the `server_name` in `gitea.conf`).
It will most likely ask you to point it to the database IP address and give it the necessary information the first time you log in.

## Managing the Gitea config

You have probably been enjoying your beautiful Gitea instance for a little bit now.
There is just one more thing we have not done!
How will you manage the Gitea configuration file?
Gitea has a configuration file called `app.ini` that you are not currently able to change, unless you SSH into the Gitea minion and change it yourself.

Fortunately there is a solution for that as well, similar to what we have done before with Nginx.
Just place the `app.ini` configuration in the Gitea directory here and make Salt sync it up and restart the Docker container when it changes.
Append the following to `gitea/init.sls`:

```yaml
manage_gitea_conf:
  file.managed:
    - name: /data/gitea/gitea/conf/app.ini
    - source: salt://gitea/app.ini
  cmd.run:
    - name: docker restart gitea
    - onchanges:
      - file: /data/gitea/gitea/conf/app.ini
```

We also need to place the `app.ini` file in `gitea/app.ini` like we have done before.
However, this file has a few things that are generated uniquely to your Gitea installation that need you need to keep.
That's why we can't give you an example file here.
You need to copy the file's contents from your Gitea minion.
This is easily done using `salt gitea cmd.run 'cat /data/gitea/gitea/conf/app.ini'`.
Salt will output the contents of this file in your terminal now.
Just copy them and place them in the file `gitea/app.ini`.
`init.sls` now checks this file and will sync your configuration to the minion and restart the Docker container if needed.

You can test if this works by changing it.
On the Gitea website, when you are logged out, look for the registration button.
Let's turn that off, so no one can register on your Gitea instance unless you create an account for them.

In `app.ini` look for the `[service]` section and set the following settings like this:

```ini
DISABLE_REGISTRATION      = true
SHOW_REGISTRATION_BUTTON  = false
```

`DISABLE_REGISTRATION` is `false` by default and `SHOW_REGISTRATION_BUTTON` is not even there by default.
Save the file and run `salt '*' state.apply` again to apply the settings.
Refresh the webpage again and you will see that the registration button is gone now!

## Conclusion

We have arrived at the end of the tutorial.
To summarize what we have done:  
We have deployed three minions: a Gitea instance, a database and a reverse proxy.
We set these minions up to run their respective application and connect to eachother to provide the full service.
We have done all of this using SaltStack.

### Cleanup

If you don't want to keep this setup and work with it, you should probably delete the instances.
EC2 instances are not cheap and four of them are certainly not cheap!
Just delete the instances and then delete the VPC from the AWS console.


