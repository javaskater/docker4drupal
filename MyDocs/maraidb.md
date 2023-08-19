## I don't want to reinstall everytime

### uncomment the volume path

* on the __docker4drupal/docker-compose.yml__
* there is:
```yaml
#    volumes:
#      - ./mariadb-init:/docker-entrypoint-initdb.d # Place init .sql file(s) here.
#      - /path/to/mariadb/data/on/host:/var/lib/mysql # Use bind mount
``` 
* I have to uncomment volumes and the second volume line
  * and for that line adapt the path on host
  * I uncomment the two path