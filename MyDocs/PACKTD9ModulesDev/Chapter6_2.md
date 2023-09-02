### Testing the tempStore
* p 144 and after
* in the phpShell
```php
> $factory = \Drupal::service('tempstore.private');
= Drupal\Core\TempStore\PrivateTempStoreFactory {#4594
    +"_serviceId": "tempstore.private",
  }

> $store = $factory->get('my_module.my_collection');
= Drupal\Core\TempStore\PrivateTempStore {#4600}

> $store->set('my_key', 'my_value');
= null

> $value = $store->get('my_key');
= "my_value"
```
* 2 things to note
  * the database connection gets away if you don't use the php drush shell for a long time
  * it is only on the __set__ command that the line is created on the __key_value_expire__ table
    * the key is a hash foloowed by the key value
    * the value is a blob which contains (not the key) 
* using the _sql:cli_ drush command whe get:
```sql
MariaDB [drupal]> SELECT name  FROM `key_value_expire` where collection='tempstore.private.my_module.my_collection'; 
+----------------------------------------------------+
| name                                               |
+----------------------------------------------------+
| Hu0bRFGpuKxt2-8wMPC0sIuq2DDZIm2i7KX8TaX3hac:my_key |
+----------------------------------------------------+
MariaDB [drupal]> SELECT value  FROM `key_value_expire` where collection='tempstore.private.my_module.my_collection'; 
+-----------------------------------------------------------------------------------------------------------------------------------------+
| value                                                                                                                                   |
+-----------------------------------------------------------------------------------------------------------------------------------------+
| O:8:"stdClass":3:{s:5:"owner";s:43:"Hu0bRFGpuKxt2-8wMPC0sIuq2DDZIm2i7KX8TaX3hac";s:4:"data";s:8:"my_value";s:7:"updated";i:1693656766;} |
+-----------------------------------------------------------------------------------------------------------------------------------------+
```
* The HAsh befoire my_key is due to the use as an anonmymous user id!
  * Internet states that by default drush is run with user 0 
* To swith to the admin (user 1) I follow that [Drupal Post](https://www.drupal.org/node/2377441) which gives
```php
> $accountSwitcher = Drupal::service('account_switcher');
= Drupal\Core\Session\AccountSwitcher {#4596
    +"_serviceId": "account_switcher",
  }

> $accountSwitcher->switchTo(new Drupal\Core\Session\UserSession(['uid'=>1])); //sees the constructor of USer Session
= Drupal\Core\Session\AccountSwitcher {#4596
    +"_serviceId": "account_switcher",
  }

> $factory = \Drupal::service('tempstore.private');
= Drupal\Core\TempStore\PrivateTempStoreFactory {#4605
    +"_serviceId": "tempstore.private",
  }

> $store = $factory->get('my_module.my_collection');
= Drupal\Core\TempStore\PrivateTempStore {#4597}

> $store->set('my_key2', 'my_value');
= null
```
* which add a new entry in the __key_value_expire__ sql table:
```sql
MariaDB [drupal]> SELECT name  FROM `key_value_expire` where collection='tempstore.private.my_module.my_collection';
+----------------------------------------------------+
| name                                               |
+----------------------------------------------------+
| 1:my_key2                                          |
| Hu0bRFGpuKxt2-8wMPC0sIuq2DDZIm2i7KX8TaX3hac:my_key |
+----------------------------------------------------+
MariaDB [drupal]> SELECT value  FROM `key_value_expire` where collection='tempstore.private.my_module.my_collection' and name LIKE '1:%'; 
+------------------------------------------------------------------------------------------+
| value                                                                                    |
+------------------------------------------------------------------------------------------+
| O:8:"stdClass":3:{s:5:"owner";i:1;s:4:"data";s:8:"my_value";s:7:"updated";i:1693660005;} |
+------------------------------------------------------------------------------------------+
```
* starting a new Session of drush php (with a different sessionid)
  * makes me unable to delete previous my_key or get the value from them because they has a different SESSION_ID (kind of user_if for the anonymous users) 

## With the Shared tmpStore

* same table but 
  * collection name starts with _tempstore.shared_
  * the key is no longer prefix by the userid (the session id in case of anonymous)
    * because everyone can use it!
* What new we can do:
  * the result true or false indicates that a change has been made or not
```php
 $store->setIfNotExists('my_key', 'my_value');
= true

> $store->setIfNotExists('my_key', 'my_value2');
= false

> $store->setIfOwner('my_key', 'my_value2');
= true
```
## User Data:

```php
> $userData = \Drupal::service('user.data');
= Drupal\user\UserData {#4597
    +"_serviceId": "user.data",
  }

> $value = ['a'=> ['b'=> 'bonjour']];
= [
    "a" => [
      "b" => "bonjour",
    ],
  ]

> $userData->set('my_module', 1, 'general', $value);
= null

> $userData->get('my_module', 1, 'general');
= [
    "a" => [
      "b" => "bonjour",
    ],
  ]
```
* ce qui donne en base:
```sql
MariaDB [drupal]> select value, serialized from users_data where uid=1;
+--------------------------------------------+------------+
| value                                      | serialized |
+--------------------------------------------+------------+
| a:1:{s:1:"a";a:1:{s:1:"b";s:7:"bonjour";}} |          1 |
+--------------------------------------------+------------+
```
* on peut supprimer ce users_data:
```php
> $userData = \Drupal::service('user.data');
= Drupal\user\UserData {#4597
    +"_serviceId": "user.data",
  }

> $userData->delete('my_module', 1, 'general');
= null
```

