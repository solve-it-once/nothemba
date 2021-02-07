# nothemba Drupal 9 instance

This readme has step-by-step instructions for installing a local version of the site and working with it.

## Prerquisites

  1. Ideally, [the CLI for platform.sh](https://docs.platform.sh/development/cli.html)
  2. An SSH keypair, with the public key added to your platform.sh account and github account. Use
    `ssh-keygen -t rsa -b 4096 -C "myname@nameofmycomputer"` to generate a new keypair
  3. PHP 7.2+ CLI on your machine
  4. [Composer](https://getcomposer.org/doc/faqs/how-to-install-composer-programmatically.md), globally
  5. Docker and docker-compose, preferably installed via a package manager, but directly is fine too
  6. git (configured with your username/email and ssh key. See below for some helpful configs)
  7. [drush](https://docs.drush.org/en/7.x/install/), preferably global on your machine, though this is included in the codebase

### Text editor or IDE

The PHPStorm IDE is really great for traversing the codebase and its dependencies, though it has an annual license fee. 
One big benefit of PHPStorm, though, is how easy it is to set up a database for viewing.

Text editors such as Atom, VS Code, Sublime Text, vim, and others are also good, and can be set up to do things like xdebug.

### /etc/hosts

In order to make the docker containers interface with the local more smoothly (which helps with connecting to them), add 
the following two lines to your `/etc/hosts` file (or equivalent):

```
0.0.0.0 mysql
0.0.0.0 solr
```

## Cloning and initializing a local site

`cd` to where you want your site to be on your computer. I like `~/htdocs`, but any folder will do. Once there, run 
`git clone git@github.com:solve-it-once/nothemba.git` to clone the repo locally. `cd` into it and you'll be able to 
traverse the codebase.

### .git/config

Whether via CLI or a filesystem UI, navigate to and open `.git/config` in the repo, and replace the entire file contents 
with:

```
[core]
  repositoryformatversion = 0
  filemode = false
  bare = false
  logallrefupdates = true
[remote "origin"]
  url = git@github.com:solve-it-once/nothemba.git
  url = jzvvphvwpflyg@git.eu-4.platform.sh:jzvvphvwpflyg.git
  fetch = +refs/heads/*:refs/remotes/origin/*
[remote "platform"]
  url = jzvvphvwpflyg@git.eu-4.platform.sh:jzvvphvwpflyg.git
  fetch = +refs/heads/*:refs/remotes/platform/*
[remote "github"]
  url = git@github.com:solve-it-once/nothemba.git
  fetch = +refs/heads/*:refs/remotes/github/*
[branch "master"]
  remote = origin
  merge = refs/heads/master

```

This setup lets your push/pull from origin to update the codebase in two places, which is what you want to do most of the 
time. There are also remotes for platform (the hosting service) and github, so you can push, pull, etc. to just one remote 
at a time when neeeded. This config file also sets filemode to false, which tends to make commits smoother and cleaner.

### web/sites/default/settings.local.php

Create a file at `web/sites/default/` called settings.local.php and put the following as its contents:

```
<?php

/**
 * Local error reporting.
 */
error_reporting(E_ALL);
ini_set('display_errors', TRUE);
ini_set('display_startup_errors', TRUE);
ini_set('xdebug.max_nesting_level', 512);
ini_set('memory_limit', '512M');
$config['system.logging']['error_level'] = 'verbose';
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;
$config['advagg.settings']['enabled'] = FALSE;

$settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';
$settings['container_yamls'][] = DRUPAL_ROOT . '/sites/default/local.services.yml';
$settings['file_chmod_directory'] = 0755;
$settings['file_chmod_file'] = 0644;
$settings['hash_salt'] = 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX';
$settings['skip_permissions_hardening'] = TRUE;
$settings['trusted_host_patterns'][] = '^localhost.*$';
$settings['trusted_host_patterns'][] = '^127\.0.*$';

$databases['default']['default'] = array (
  'database' => 'drupal',
  'username' => 'root',
  'password' => 'root',
  'prefix' => '',
  'host' => 'mysql',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
);

$config['search_api.server.solr']['backend_config']['connector_config']['core'] = 'd8';
$config['search_api.server.solr']['backend_config']['connector_config']['path'] = '';
$config['search_api.server.solr']['backend_config']['connector_config']['host'] = 'solr';
$config['search_api.server.solr']['backend_config']['connector_config']['port'] = '8983';

// Configure private and temporary file paths.
if (!isset($settings['file_private_path'])) {
  $settings['file_private_path'] = '../private';
}

assert_options(ASSERT_ACTIVE, TRUE);
\Drupal\Component\Assertion\Handle::register();

```

These are the sorts of things that set up Drupal for your local development machine. There's lots of logging and suppression 
of production config values, plus necessary connection information for Drupal to interface with the Docker-based mysql and 
Solr servers.

### Setup steps

Once you have the requisite files in place per above, setting up a local site should involve only:

  1. Run `docker-compose build --no-cache` to initialize the docker containers
  2. Run `docker-compose up` to launch the containers. You'll need a new terminal tab/window since the process runs
    as long as you're working on the local (`Ctrl+C` in that terminal tab to stop)
  3. Run `composer install` to install the dependencies
  4. Check that the configs above are in place. First run `git remote` to ensure you see origin, platform, and github. Then 
    run `drush sql-connect` (the rest of this readme assumes global drush. If not, run `./vendor/bin/drush` in its stead) 
    to see the command drush uses to run mysql. This command should succeed, and should have the db information from the 
    local settings.local.php file
  5. We'll want to get the database and files from the server environment. There's two ways to do this:
    1. Use the Platform.sh CLI by running `platform mount:download` and choosing to download all mounts when prompted. For 
      the database, run `platform db:dump` to download a file containing the database. You can then run 
      ```
        `drush sql-connect` < downloaded-file-name.sql
      ``` to import it to the mysql server container
    2. With drush you may be able to run `drush sql-sync @nothemba.platformmaster @self` to bring the database down, and 
      `drush rsync @nothemba.platformmaster:%files @self:%files` to get the files
  6. Visit http://localhost:8000/ to see your local site!

## Common operations

Here's a list of common commands in drush and otherwise that come in handy for working with the local site:

  * `drush cr` - clears all the caches
  * `drush updatedb` - runs database updates that are pending
  * `drush uli` gives you a login link as the admin user
  * `drush cex` - exports the configuration in the database down to yaml files in the codebase
  * `drush cim` - imports configuration yamls into the database
  * `composer update` - updates drupal core and its dependencies, which is especially useful when there a security patch

## Performing routine Drupal updates

Whether for maintenance or to ensure all Security patches are up-to-date, the following procedure can be used to perform
software updates:

  1. `composer install`
  2. `composer update`
  3. `drush updatedb`
  4. `drush cr`
  5. (Test the site on your local to ensure it looks and works as intended prior to dev-level QA)
  6. `drush cex`
  7. `git add .`
  8. `git commit -m "composer update (Drupal core and contrib updates)"`
  9. `git push origin master`
