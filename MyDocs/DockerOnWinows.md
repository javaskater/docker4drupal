## Starting my project in WSL

### without starting Docker on windows

```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make up
Starting up containers for my_drupal9_project...
docker-compose pull

The command 'docker-compose' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.

For details about using Docker Desktop with WSL 2, visit:

https://docs.docker.com/go/wsl2/

make: *** [docker.mk:22: up] Error 1
```

### Docker integration with WSL Ubuntu

* How to tell Docker for Windows to run on WSL2 Linux Enginr see [this chapter of the official documentation](https://docs.docker.com/desktop/wsl/#turn-on-docker-desktop-wsl-2)
* I Upadated Docker for Windows
* I restarted the images:
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make up
Starting up containers for my_drupal9_project...
docker-compose pull
[+] Pulling 6/6
 ✔ php Skipped - Image is already being pulled by crond                                                                                  0.0s 
 ✔ crond Pulled                                                                                                                          4.3s 
 ✔ mailhog Pulled                                                                                                                        3.1s 
 ✔ traefik Pulled                                                                                                                        4.1s 
 ✔ mariadb Pulled                                                                                                                        3.3s 
 ✔ nginx Pulled                                                                                                                          3.5s 
docker-compose up -d --remove-orphans
[+] Running 6/6
 ✔ Container my_drupal9_project_traefik  Started                                                                                         2.4s 
 ✔ Container my_drupal9_project_crond    Started                                                                                         1.9s 
 ✔ Container my_drupal9_project_mailhog  Started                                                                                         1.9s 
 ✔ Container my_drupal9_project_php      Started                                                                                         2.3s 
 ✔ Container my_drupal9_project_mariadb  Started                                                                                         2.3s 
 ✔ Container my_drupal9_project_nginx    Started                                                                                         2.5s 
``` 
* A the end of the start I find my 2 mounted mariadb volumes
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ ls -ltr | tail -1
drwxr-xr-x 2 root   root    4096 Aug 17 17:04 mariadb-init
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ ls -ltr .. | tail -1
drwxr-xr-x 6 systemd-network systemd-journal 4096 Aug 17 17:04 mysql
```
 ### Accessing Drupal and its database using Firefox:

 * The traefik tags are specified container by container in each __docker-compose.yml__ and __docker-compose.override.yml__
 * for the NGINX container it is in 
  ```yaml
  labels:
    - "traefik.http.routers.${}_nginx.rule=Host(`${PROJECT_BASE_URL}`)"
  ```

* with the two variables __PROJECT_NAME__ and __PROJECT_BASE_URL__ defined in the __.env__ file:
```bash
PROJECT_NAME=my_drupal9_project
PROJECT_BASE_URL=drupal.docker.localhost
```
* __PROJECT_PORT__ is the port exposed by Traefik for our nginx
```yaml
  traefik:
    image: traefik:v2.0
    container_name: "${PROJECT_NAME}_traefik"
    command: --api.insecure=true --providers.docker
    ports:
    - "${PROJECT_PORT}:80"
#    - '8080:8080' # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
* its value is in the .env file:
```bash
PROJECT_PORT=8000
```

### accessing  Drupal:
* we access the port 80 of NGINX through this URL [drupal.docker.localhost:8000](http://drupal.docker.localhost:8000)
  * in this downloaded version of the version du projet Github project, it is the  9.4.8 Drupal's version which get installed
  * the user/password of the admin user is _d8igpde1/d8igpde1_
  * problem Traefik on port 8000 does not transmit css and js files

* Log on stdout by starting the containers without the -d :
```bash
my_drupal9_project_nginx    | 2023/08/18 17:44:47 [error] 49#49: *5 open() "/var/www/html/web/sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js" failed (2: No such file or directory), client: 172.18.0.7, server: default, request: "GET /sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js HTTP/1.1", host: "drupal.docker.localhost:8000", referrer: "http://drupal.docker.localhost:8000/"
```

 * the file exists and can be read by everyone on the php container
```bash
wodby@php.container:/var/www/html $ ls -l /var/www/html/web/sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js
-rw-rw-r--    1 www-data www-data       310 Aug 18 14:47 /var/www/html/web/sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js
```
* What on the nginx container ?
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make shell nginx
docker exec -ti -e COLUMNS=142 -e LINES=8 1fa982605502 sh
/var/www/html $ ls -l /var/www/html/web/sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js
ls: /var/www/html/web/sites/default/files/js/js_pJBs_U5CFeW43rfMO4MmmpBhEM0fX5cxZigDLLHuc5Q.js: No such file or directory
# it is empty because it is cached ????
/var/www/html $ ls -l web/sites/default/files/
total 0
#the user is wodby
/var/www/html $ ls -l web/sites/default/
total 88
-rw-r--r--    1 wodby    wodby         8277 Nov 30  2022 default.services.yml
-rw-r--r--    1 wodby    wodby        32904 Nov 30  2022 default.settings.php
drwxrwxrwx    2 wodby    wodby         4096 Aug 18 17:00 files
-rw-r--r--    1 wodby    wodby        33694 Feb  9  2023 settings.php
```
#### The solved problem
* in the _docker-compose.override.yml_ we had another code link:
  * the html folder was sym-linked with the parent folder
* I changed it back (like in the original Github file)
```yaml
services:
  php:
    image: wodby/drupal:$DRUPAL_TAG
    environment:
      PHP_FPM_CLEAR_ENV: "no"
    volumes:
    - ./:/var/www/html:cached
```
* the same thing applies to the _nginx_ configuration in the same file
```yaml
nginx:
    volumes:
    - ./:/var/www/html:cached
```
* first call a cache rebuild (because in the files folder has only the php cache)
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make drush cr
docker exec 7dba836a10e7 drush -r /var/www/html/web cr
 [success] Cache rebuild complete.
```
* now on the nginx container
```bash
/var/www/html $ ls -l web/sites/default/files/css
total 148
-rw-rw-r--    1 82       www-data     56610 Aug 19 10:37 css_22RgfQqO_I_o0o5PqJFhkNg7XSSeQKm_cTbAm95ysaE.css
-rw-rw-r--    1 82       www-data     10859 Aug 19 10:37 css_22RgfQqO_I_o0o5PqJFhkNg7XSSeQKm_cTbAm95ysaE.css.gz
-rw-rw-r--    1 82       www-data     13631 Aug 19 10:37 css_3wBWe5HjtU919CDHljgYi1m7M-oagwF27SuSlsQ_vuU.css
-rw-rw-r--    1 82       www-data      3307 Aug 19 10:37 css_3wBWe5HjtU919CDHljgYi1m7M-oagwF27SuSlsQ_vuU.css.gz
-rw-rw-r--    1 82       www-data      2044 Aug 19 10:37 css_A4fAF-tSxQQaqdlU6gtvXxawnLnm1r9h90dUSEe-4eY.css
-rw-rw-r--    1 82       www-data       723 Aug 19 10:37 css_A4fAF-tSxQQaqdlU6gtvXxawnLnm1r9h90dUSEe-4eY.css.gz
-rw-rw-r--    1 82       www-data     35596 Aug 19 10:37 css_TIXkfFy9OOnQFxt4W2cV5lcSBSq5_nvlPQkuzhaAJ1k.css
-rw-rw-r--    1 82       www-data      6261 Aug 19 10:37 css_TIXkfFy9OOnQFxt4W2cV5lcSBSq5_nvlPQkuzhaAJ1k.css.gz
-rw-rw-r--    1 82       www-data       509 Aug 19 10:37 css_Z5jMg7P_bjcW9iUzujI7oaechMyxQTUqZhHJ_aYSq04.css
-rw-rw-r--    1 82       www-data       274 Aug 19 10:37 css_Z5jMg7P_bjcW9iUzujI7oaechMyxQTUqZhHJ_aYSq04.css.gz
```
#### Why does it try to access http://example.com/sites/default/files/css/...?
* I solved this problem by Emptying the cookies of Firefox
  * Parameters / Privacy (Tab) / Button Erase under the Cookies chapter 
* I had to log again as admin through the URL [drupal.docker.localhost:8000/user/login](http://drupal.docker.localhost:8000/user/login)

#### accessing Drupal throupgh private IP address

* [This article](https://linuxhandbook.com/get-container-ip/) explain well how we access the container IP
  * [That Stackoverflow post](https://stackoverflow.com/questions/17157721/how-to-get-a-docker-containers-ip-address-from-the-host) gives a functioning answer 
* For me it is the nginx container I want to access
  * getting the container id or name
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ make ps | grep -i nginx
1fa982605502   wodby/nginx:1.23-5.27.0     "/docker-entrypoint.…"   26 hours ago   Up 59 seconds       80/tcp                 my_drupal9_project_nginx
```
* getting the corresponding private IP address:
```bash
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ docker container inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}
' 1fa982605502
172.18.0.8
jpmena@LAPTOP-E2MJK1UO:~/CONSULTANT/Docker/docker4drupal$ docker container inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my_drupal9_project_nginx
172.18.0.8
```
* but no network for those private addresses on WSL I can't access them directly


### accessing the mariadb database through phpmyadmin
* for the phpmyadmin container it is:
  * first we need to remove the comments to take this container into account
  * stop and starts all the containers
      * the new phpmyadmin image takes some time to be downloaded
      * for that container the Traefik rule is
  ```yaml
    labels:
    - "traefik.http.routers.${PROJECT_NAME}_pma.rule=Host(`pma.${PROJECT_BASE_URL}`)"
  ```

* We then access to the port 80 of PHPMYADMIN through the url [drupal.docker.localhost:8000](http://drupal.docker.localhost:8000)

## TODO
* how to access the Trefik admin panel ?
* How to access the containers log
  * A way to do it is to call _docker-compose up_ without the -d option 