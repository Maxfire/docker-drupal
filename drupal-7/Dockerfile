FROM ubuntu:14.04
MAINTAINER Alejandro Gomez <agommor@gmail.com>

# Arguments
ARG MYSQL_ROOT_PASSWORD=admin
ARG DRUPAL_ADMIN_PASSWORD=admin
ARG SSH_ROOT_PASSWORD=root
ARG DRUPAL_VERSION=7.50

# Environments args
ENV DEBIAN_FRONTEND noninteractive
ENV MYSQL_ROOT_PASSWORD ${MYSQL_PASSWORD}
ENV DRUPAL_ADMIN_PASSWORD ${DRUPAL_ADMIN_PASSWORD}
ENV SSH_ROOT_PASSWORD ${SSH_ROOT_PASSWORD}
ENV DRUPAL_VERSION ${DRUPAL_VERSION}

RUN rm /bin/sh && ln -s /bin/bash /bin/sh

# Setup MySQL (before the installation).
RUN debconf-set-selections <<< "mysql-server mysql-server/root_password password ${MYSQL_ROOT_PASSWORD}"
RUN debconf-set-selections <<< "mysql-server mysql-server/root_password_again password ${MYSQL_ROOT_PASSWORD}"

# Install packages.
RUN apt-get update
RUN apt-get install -y \
	vim \
	git \
	apache2 \
	php5-cli \
	php5-mysql \
	php5-gd \
	php5-curl \
	php5-xdebug \
	libapache2-mod-php5 \
	curl \
	mysql-server \
	mysql-client \
	openssh-server \
	phpmyadmin \
	wget \
	unzip \
	supervisor \
  sendmail
RUN apt-get clean

# Install Composer.
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer

# Install Drush 7.
RUN composer global require drush/drush:7.*
RUN composer global update
# Unfortunately, adding the composer vendor dir to the PATH doesn't seem to work. So:
RUN ln -s /root/.composer/vendor/bin/drush /usr/local/bin/drush

# Setup PHP.
RUN sed -i 's/display_errors = Off/display_errors = On/' /etc/php5/apache2/php.ini
RUN sed -i 's/display_errors = Off/display_errors = On/' /etc/php5/cli/php.ini

# Setup Apache.
# In order to run our Simpletest tests, we need to make Apache
# listen on the same port as the one we forwarded. Because we use
# 8080 by default, we set it up for that port.
RUN sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
RUN sed -i 's/DocumentRoot \/var\/www\/html/DocumentRoot \/var\/www/' /etc/apache2/sites-available/000-default.conf
RUN echo "Listen 8080" >> /etc/apache2/ports.conf
RUN sed -i 's/VirtualHost \*:80/VirtualHost \*:\*/' /etc/apache2/sites-available/000-default.conf
RUN a2enmod rewrite

# Setup PHPMyAdmin
RUN echo -e "\n# Include PHPMyAdmin configuration\nInclude /etc/phpmyadmin/apache.conf\n" >> /etc/apache2/apache2.conf
# db access
RUN sed -i -e "s/$dbuser=''/$dbuser='root'/g" /etc/phpmyadmin/config-db.php
RUN sed -i -e "s/$dbpass=''/$dbpass='${MYSQL_ROOT_PASSWORD}'/g" /etc/phpmyadmin/config-db.php

RUN sed -i -e "s/\$cfg\['Servers'\]\[\$i\]\['\(table_uiprefs\|history\)'\].*/\$cfg\['Servers'\]\[\$i\]\['\1'\] = false;/g" /etc/phpmyadmin/config.inc.php

# Setup MySQL, bind on all addresses.
RUN sed -i -e 's/^bind-address\s*=\s*127.0.0.1/#bind-address = 127.0.0.1/' /etc/mysql/my.cnf

# Setup SSH.
RUN echo "root:${SSH_ROOT_PASSWORD}" | chpasswd
RUN sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN mkdir /var/run/sshd && chmod 0755 /var/run/sshd
RUN mkdir -p /root/.ssh/ && touch /root/.ssh/authorized_keys
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

# Setup Supervisor.
RUN echo -e '[program:apache2]\ncommand=/bin/bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"\nautorestart=true\n\n' >> /etc/supervisor/supervisord.conf
RUN echo -e '[program:mysql]\ncommand=/usr/bin/pidproxy /var/run/mysqld/mysqld.pid /usr/sbin/mysqld\nautorestart=true\n\n' >> /etc/supervisor/supervisord.conf
RUN echo -e '[program:sshd]\ncommand=/usr/sbin/sshd -D\n\n' >> /etc/supervisor/supervisord.conf

# Setup XDebug.
RUN echo "xdebug.max_nesting_level = 300" >> /etc/php5/apache2/conf.d/20-xdebug.ini
RUN echo "xdebug.max_nesting_level = 300" >> /etc/php5/cli/conf.d/20-xdebug.ini

# Install Drupal.
RUN rm -rf /var/www
RUN cd /var && \
	drush dl drupal-${DRUPAL_VERSION} && \
	mv /var/drupal* /var/www
RUN mkdir -p /var/www/sites/default/files && \
	chmod a+w /var/www/sites/default -R && \
	mkdir /var/www/sites/all/modules/contrib -p && \
	mkdir /var/www/sites/all/modules/custom && \
	mkdir /var/www/sites/all/themes/contrib -p && \
	mkdir /var/www/sites/all/themes/custom && \
	chown -R www-data:www-data /var/www/
RUN /etc/init.d/mysql start && \
	cd /var/www && \
	drush si -y minimal --db-url=mysql://root:${MYSQL_ROOT_PASSWORD}@localhost/drupal --account-pass=${DRUPAL_ADMIN_PASSWORD} && \
	drush dl admin_menu devel && \
	drush en -y admin_menu simpletest devel && \
	drush vset "admin_menu_tweak_modules" 1 && \
	drush vset "admin_theme" "seven" && \
	drush vset "node_admin_theme" 1

EXPOSE 80 3306 22
CMD exec supervisord -n
