---
layout: page
title: "apache2 mod_rails recipe"
date: 2008-10-06 01:12
comments: true
categories: [apache, mod_rails, puppet]
---
I recently had to write a recipe to set up instances of mod_rails apps for apache2.  The goal of this process is to have a server that has apps hosted at http://server/app1, http://server/app2, etc.

The recipe is going to be used to create a puppet script, but it could also be boiled down to a ruby or shell script to easily create app instances for anyone's use.
<!--more--> 
### Assumptions and Server Setup ###

* All of this is done as sudo or root
* The OS has apache2, mysql, sqlite3, ruby, rubygems, git-core and
subversion installed
* The gems sqlite3-ruby, mysql, rails, capistrano are installed

If apache2 is not yet configured with mod_rails, do the following:

* Create `/etc/apache2/mods-available/mod_rails.conf` and put the following in it:

```
 # Ruby Passenger Config
 LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-2.0.3/ext/apache2/mod_passenger.so
 PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-2.0.3
 PassengerRuby /usr/bin/ruby1.8
```

* Symlink it into enabled:  `ln -s /etc/apache2/mods-available/mod_rails.conf /etc/apache2/mods-enabled/mod_rails.conf`
* Restart apache2

Optionally make Ubuntu's username policy more sensible (allow underscores)

```
$ echo "NAME_REGEX=\"[a-z_][a-z0-9_-]*[$]?\"" >> /etc/adduser.conf
```                     

### Creating a ModRails Application @http://some.host.name/appname ###

* create the user, the appname will be the username:

```
$ adduser <username> (use pwgen or something for the password)
```

* create a stub rails app:

```
$ rails /home/<username>/rails/testapp 
```

* Link the testapp to the "static" application location

```
$ ln -s /home/<username>/rails/testapp /home/<username>/rails/app
```

* Link the "static" application location into the servers webroot

```
$ ln -s /home/<username>/rails/app/public /var/www/<username>
```

* Create the apache2 conf for the rails app

```
$ echo "RailsBaseURI /<username>" > /etc/apache2/sites-available/<username>
```

* Tell apache2 to actually use the app

```
$ ln -s /etc/apache2/sites-available/<username> /etc/apache2/sites-enabled/<username>
```

* Make sure that the new user has ownership of the files

```
$ chown -R /home/<username>/rails <username>.<username>
```

* Restart apache2 and check to see if it works

```
$ apache2ctl graceful
$ wget http://localhost/<username> --server-response --spider # this should return 200 OK
```

* Create a mysql user and databases for the app

```
 $ for db in production test development; do mysql -u root -pMYSQLPASS -e "grant all privileges on <username>_$db.* to '<username>'@'localhost' identified by '<userpassword>'; create database <username>_$db;"  done
```

* Copy template capistrano deployment files into their `home/<username>/rails` directory

At this point the application will be available running at `http://hostname/<username>`

With one of the `deploy.rb` files, a savvy rails user can deploy their application to the server with capistrano

A less savvy rails user can connect via ssh and checkout their their rails app to someplace in their home, then symlink it to the `~/rails/app` directory
