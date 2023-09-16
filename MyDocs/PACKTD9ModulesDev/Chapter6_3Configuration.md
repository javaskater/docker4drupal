# Configuration API
 * It is not specific to one environment (contrary to the State the TempStore and the UserData API)
## Devel dependencies
 * marking modules for development only happens in the composer.json file and alse in the settings.php see that [drupal link](https://www.drupal.org/docs/contributed-modules/git-info-report/setting-it-up-only-on-development)
 * Example of the [Examples Module](https://www.drupal.org/project/examples)
   * I am running a Drupal 9.4.8 so the command is
   * the --dev option does not work as make composer command, go first to the php container (make shell)
```bash
wodby@php.container:/var/www/html $ composer require 'drupal/examples:^4.0' --dev
./composer.json has been updated
Running composer update drupal/examples
Loading composer repositories with package information
Updating dependencies
Lock file operations: 1 install, 0 updates, 0 removals
  - Locking drupal/examples (4.0.2)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 1 install, 0 updates, 0 removals
  - Installing drupal/examples (4.0.2): Extracting archive
1 package suggestions were added by new dependencies, use `composer suggest` to see details.
Package doctrine/reflection is abandoned, you should avoid using it. Use roave/better-reflection instead.
Package symfony/debug is abandoned, you should avoid using it. Use symfony/error-handler instead.
Generating autoload files
48 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
Found 4 security vulnerability advisories affecting 4 packages.
Run composer audit for a full list of advisories.
```
* drupal/examples is at the very end of the composer.json on the require-dev part created
```json
    "require-dev": {
        "drupal/examples": "^4.0"
    }
```
* use --no-dev for composer requiring the modules for production

* the [drupal doc page](https://www.drupal.org/docs/contributed-modules/git-info-report/setting-it-up-only-on-development) 
  * asks for adding the following code if you do not want to share the configuration of the examples module (to a production website for example)
```php
$settings['config_exclude_modules'] = ['examples'];
```
* the module anyway is a standard contrib module:
```bash
wodby@php.container:/var/www/html $ ls -l  web/modules/contrib/examples/
total 96
-rw-r--r--    1 wodby    wodby         2019 Mar  1  2023 CONTRIBUTING.md
-rw-r--r--    1 wodby    wodby        18092 Nov 16  2016 LICENSE.txt
-rw-r--r--    1 wodby    wodby         3431 Mar  1  2023 README.md
-rw-r--r--    1 wodby    wodby         3176 Mar  1  2023 STANDARDS.md
-rw-r--r--    1 wodby    wodby         4672 Mar  1  2023 TESTING.md
-rw-r--r--    1 wodby    wodby          724 Mar  1  2023 composer.json
drwxr-xr-x    2 wodby    wodby         4096 Mar  1  2023 css
-rw-r--r--    1 wodby    wodby          324 Mar  1  2023 examples.info.yml
-rw-r--r--    1 wodby    wodby           86 Mar  1  2023 examples.libraries.yml
-rw-r--r--    1 wodby    wodby         5089 Mar  1  2023 examples.module
drwxr-xr-x    2 wodby    wodby         4096 Mar  1  2023 images
drwxr-xr-x   35 wodby    wodby         4096 Mar  1  2023 modules
-rw-r--r--    1 wodby    wodby        13932 Mar  1  2023 phpcs.xml.dist
drwxr-xr-x    3 wodby    wodby         4096 Mar  1  2023 src
drwxr-xr-x    4 wodby    wodby         4096 Mar  1  2023 tests
```

### problem with activating a module

```bash
wodby@php.container:/var/www/html/web/sites/default $ ls -l
total 88
-rw-r--r--    1 wodby    wodby         8277 Nov 30  2022 default.services.yml
-rw-r--r--    1 wodby    wodby        32904 Nov 30  2022 default.settings.php
drwxrwxrwx    6 wodby    wodby         4096 Aug 19 14:32 files
-rw-r--r--    1 wodby    wodby        33455 Sep 16 14:38 settings.php
wodby@php.container:/var/www/html/web/sites/default $ cd files/
wodby@php.container:/var/www/html/web/sites/default/files $ ls -l
total 16
drwxrwxr-x    2 www-data www-data      4096 Aug 21 14:52 css
drwxrwxr-x    2 www-data www-data      4096 Sep 16 14:09 js
drwxrwxr-x    2 www-data www-data      4096 Aug 19 14:58 languages
drwxrwxrwx    3 www-data www-data      4096 Aug 21 14:53 php
wodby@php.container:/var/www/html/web/sites/default/files $ mkdir translations && chmod 777 translations
```
* module activation now works even from the host (makefile):
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make drush en block_example
docker exec 7dba836a10e7 drush -r /var/www/html/web en block_example
The following module(s) will be enabled: block_example, examples

 Do you want to continue? (yes/no) [yes]:
 > 
>  [notice] Traduction fr vérifiée pour examples.
>  [notice] Traduction fr téléchargée pour examples.
>  [notice] Traduction fr importée pour examples.
>  [notice] Translations imported: 0 added, 141 updated, 0 removed.
>  [warning] No configuration objects have been updated.
>  [notice] Message: 1 fichier de traduction importé. /0/ traductions ont été ajoutées, /141/ 
> traductions ont été mises à jour et /0/ traductions ont été supprimées.
> 
>  [notice] Message: Aucun objet de configuration n'a été mis à jour.
> 
 [success] Successfully enabled: block_example, examples
```
* the translation file is at the right place:
```bash
wodby@php.container:/var/www/html $ ls -l web/sites/default/files/translations/
total 8
-rw-rw-r--    1 wodby    wodby         6423 Sep 16 14:50 examples-4.0.2.fr.po
```

## How to define your sync directory:

* following that [Drupal page](https://www.drupal.org/docs/configuration-management/changing-the-storage-location-of-the-sync-directory)
* you can place your sync directory higher than the web directory to have it in your Git control version system
* on my [Docker4Drupal](https://github.com/wodby/docker4drupal) it is by default:
```php
$settings['config_sync_directory'] = '/mnt/files/config/sync_dir';
```
* What is the _/mnt/files_?
```bash
wodby@php.container:/var/www/html $ ls -l /mnt/files/config/sync_dir
total 0 #it exists but is empty
wodby@php.container:/var/www/html $ ls -l /mnt/files/config/
total 4
drwxr-xr-x    2 www-data www-data      4096 Aug 19 14:32 sync_dir #it has not the right permissions for drush (wodby) it is meant for  use by the graphical web interface
```
* following that [Issue](https://github.com/wodby/docker4drupal/issues/240) I redifine it upper in the hierarchy
```php
#$settings['config_sync_directory'] = '/mnt/files/config/sync_dir';
$settings['config_sync_directory'] = '../config/sync';
```
* It now works (even using the make drush command from the host)
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make drush config:export
docker exec 7dba836a10e7 drush -r /var/www/html/web config:export
 [success] Configuration successfully exported to ../config/sync.
../config/sync 
```
* it creates the directories
* you can call it as many times as you want