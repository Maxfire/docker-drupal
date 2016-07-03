Drupal 7 development with Docker
==============================

Quick and easy to use Docker container for your *local Drupal 7 development*. It contains a LAMP stack and an SSH server, along with an up to date version of Drush. It is based on [Wouter Admiraal](https://github.com/wadmiraal/docker-drupal) but using an **[Ubuntu 14.04](https://hub.docker.com/_/ubuntu/)** as OS.

Summary
-------

This image contains:

* Apache 2.4
* MySQL 5.5
* PHP 5.6
* Drush 7
* Drupal 7.44 by default but another 7.X version can be installed.
* Composer
* PHPMyAdmin
* Blackfire

When launching, the container will contain a fully-installed, ready to use Drupal site.

### Passwords

By defaults, the passwords are:

* Drupal: `admin:admin`
* MySQL: `root:` (no password)
* SSH: `root:root`
 
Anyway, the passwords can be defined (see the installation section).

### Exposed ports

* 80 (Apache)
* 22 (SSH)
* 3306 (MySQL)

Installation
------------

### Github

This is the most powerful approach: you can customize your drupal docker image easily!

Clone the repository locally and build it:

	git clone https://github.com/agomezmoron/docker-drupal.git
	cd docker-drupal-7
	docker build -t yourname/drupal7 .
	
**Important:** If your docker version is <1.9, so you will have to edit the [Dockerfile](Dockerfile) removing the ARG sections.

You can define some passwords (in case you want to have an image for production, for example). To do that you only has to set the variables in the docker build command (docker 1.9+):
	
	docker build  build --build-arg MYSQL_ROOT_PASSWORD=admin,DRUPAL_ADMIN_PASSWORD=admin,SSH_ROOT_PASSWORD=root,DRUPAL_VERSION=7.44  -t yourname/drupal8 .

### Docker repository

Get the image:

The provided image is coming with the default versions described before.

	docker pull agomezmoron/drupal7 (pending to be uploaded)

#### Tags

Nowadays the only tag is the latest one.

Running it
----------

For optimum usage, map some local directories to the container for easier development. I personally create at least a `modules/` directory which will contain my custom modules. You can do the same for your themes.

The container exposes its `80` port (Apache), its `3306` port (MySQL) and its `22` port (SSH). Make good use of this by forwarding your local ports. You should at least forward to port `80` (using `-p local_port:80`, like `-p 8080:80`). A good idea is to also forward port `22`, so you can use Drush from your local machine using aliases, and directly execute commands inside the container, without attaching to it.

Here's an example just running the container and forwarding `localhost:8080` and `localhost:8022` to the container:

	docker run -d -p 8080:80 -p 8022:22 -t agomezmoron/drupal7

### Writing code locally

Here's an example running the container, forwarding port `8080` like before, but also mounting Drupal's `sites/all/modules/custom/` folder to my local `modules/` folder. I can then start writing code on my local machine, directly in this folder, and it will be available inside the container:

	docker run -d -p 8080:80 -v `pwd`/modules:/var/www/sites/all/modules/custom -t agomezmoron/drupal7

### Using Drush

Using Drush aliases, you can directly execute Drush commands locally and have them be executed inside the container. Create a new aliases file in your home directory and add the following:

	# ~/.drush/docker.aliases.drushrc.php
	<?php
	$aliases['agomezmoron_drupal'] = array(
	  'root' => '/var/www',
	  'remote-user' => 'root',
	  'remote-host' => 'localhost',
	  'ssh-options' => '-p 8022', // Or any other port you specify when running the container
	);

Next, if you do not wish to type the root password everytime you run a Drush command, copy the content of your local SSH public key (usually `~/.ssh/id_rsa.pub`; read [here](https://help.github.com/articles/generating-ssh-keys/) on how to generate one if you don't have it). SSH into the running container:

	# If you forwarded another port than 8022, change accordingly.
	# Password is "root" by default
	ssh root@localhost -p 8022

Once you're logged in, add the contents of your `id_rsa.pub` file to `/root/.ssh/authorized_keys`. Exit.

You should now be able to call:

	drush @docker.agomezmoron_drupal cc all

This will clear the cache of your Drupal site. All other commands will function as well.

### Running tests

If you want to run tests, you may need to take some additional steps. Drupal's Simpletest will use cURL to simulate user interactions with a freshly installed site when running tests. This "virtual" site resides under `http://localhost:[forwarded ip]`. This gives issues, though, as the *container* uses port `80`. By default, the container's virtual host will actually listen to *any* port, but you still need to tell Apache on which ports it should bind. By default, it will bind on `80` *and* `8080`, so if you use the above examples, you can start running your tests straight away. But, if you choose to forward to a different port, you must add it to Apache's configuration and restart Apache. You can simply do the following:

	# If you forwarded to another port than 8022, change accordingly.
	# Password is "root" by default
	ssh root@localhost -p 8022
	# Change the port number accordingly. This example is if you forward
	# to port 8081.
	echo "Listen 8081" >> /etc/apache2/ports.conf
	/etc/init.d/apache2 restart

Or, shorthand:

	ssh root@localhost -p 8022 -C 'echo "Listen 8081" >> /etc/apache2/ports.conf && /etc/init.d/apache2 restart'

### MySQL and PHPMyAdmin

PHPMyAdmin is available at `/phpmyadmin`. The MySQL port `3306` is exposed. The root account for MySQL is `root` (no password by default).
