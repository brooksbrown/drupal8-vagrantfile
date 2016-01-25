# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.network "private_network", ip: "192.168.33.10"

  config.vm.provider :virtualbox do |vb|
    vb.memory = 1024
  end

  config.vm.provision "shell", inline: <<-SHELL
    SITENAME=test
    SITE_DOMAIN=192.168.33.10
    TEST_DB_NAME=test
    TEST_DB_USER=vagrant
    TEST_DB_PASS=vagrant
    MYSQL_ROOT_PASS=vagrant
    
    apt-get update
    apt-get install -y debconf git

    # LAMP Stack
    #
    echo "mysql-server-5.5 mysql-server/root_password_again password $MYSQL_ROOT_PASS" | debconf-set-selections
    echo "mysql-server-5.5 mysql-server/root_password password $MYSQL_ROOT_PASS" | debconf-set-selections
    apt-get install -y lamp-server^
    apt-get install -y mysql-client php5-dev php5-gd php5-curl
    pecl install uploadprogress
    
    if ! grep -Fxq "extension=uploadprogress.so" /etc/php5/apache2/php.ini; then
      echo "extension=uploadprogress.so" >> /etc/php5/apache2/php.ini
    fi

    a2enmod rewrite
    a2enmod headers

    # allow htaccess files
    sed -i "s/AllowOverride None/AllowOverride All/" /etc/apache2/apache2.conf

    service apache2 restart


    # Test database creation
    #
    if mysql -u $TEST_DB_USER -p$TEST_DB_PASS -e "use $TEST_DB_NAME"; then
      mysql -u root -p$MYSQL_ROOT_PASS -e "drop database $TEST_DB_NAME"
    fi
    mysql -u root -p$MYSQL_ROOT_PASS -e "create database $TEST_DB_NAME"
    mysql -u root -p$MYSQL_ROOT_PASS -e "grant usage on $TEST_DB_NAME.* to $TEST_DB_USER@localhost identified by '$TEST_DB_PASS'"
    mysql -u root -p$MYSQL_ROOT_PASS -e "grant all privileges on $TEST_DB_NAME.* to $TEST_DB_USER@localhost"
    mysql -u root -p$MYSQL_ROOT_PASS -e "flush privileges"

    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer

    curl -sSO http://files.drush.org/drush.phar
    chmod +x drush.phar
    mv drush.phar /usr/local/bin/drush
    drush init

    # Drupal initialization 
    #
    echo "Downloading drupal8..."
    if [ -d "/home/vagrant/drupal8" ]; then
      rm -rf /home/vagrant/drupal8
    fi

    drush dl drupal
    mv drupal-* /home/vagrant/drupal8
    mkdir -p /home/vagrant/drupal8/sites/$SITENAME/files 
    cp /home/vagrant/drupal8/sites/default/default.settings.php /home/vagrant/drupal8/sites/$SITENAME/settings.php
    cp /home/vagrant/drupal8/sites/example.sites.php /home/vagrant/drupal8/sites/sites.php
    echo "\\$sites['$SITE_DOMAIN'] = '$SITENAME';" >> /home/vagrant/drupal8/sites/sites.php
    echo "\\$databases['default']['default'] = array('driver' => 'mysql', 'database' => '$TEST_DB_NAME', 'username' => '$TEST_DB_USER', 'password' => '$TEST_DB_PASS', 'host' => 'localhost');" >> /home/vagrant/drupal8/sites/test/settings.php
    chown -R vagrant:vagrant /home/vagrant/drupal8
    chown -R www-data:www-data /home/vagrant/drupal8/sites/$SITENAME/files

    rm -r /var/www/html
    cd /var/www && ln -s /home/vagrant/drupal8 html && cd -

    if [ ! -d "/vagrant/modules" ]; then
      mkdir -p /vagrant/modules
    fi

    cd /home/vagrant/drupal8 && rm -r modules && ln -s /vagrant/modules . && cd -

    echo "installing drupal..."
    cd /home/vagrant/drupal8/sites/$SITENAME && drush site-install -y && cd -
    chown -R www-data:www-data /home/vagrant/drupal8/sites/$SITENAME/files
  SHELL

end
