# The chapter starts at page 141
## The state API
* in the contructor of _docker4drupal/web/core/lib/Drupal/Core/State/State.php_
```php
  /**
   * Constructs a State object.
   *
   * @param \Drupal\Core\KeyValueStore\KeyValueFactoryInterface $key_value_factory
   *   The key value store to use.
   */
  public function __construct(KeyValueFactoryInterface $key_value_factory) {
    $this->keyValueStore = $key_value_factory->get('state');
  }
```
* so passing the database request on [phpMyAdmin](http://pma.drupal.docker.localhost:8000) gives a lot of results:
```sql
SELECT * FROM `key_value` where collection = 'state'; 
```

### Test of the state API

* We cannot access the Drush php console through make
* we are obliged to access it through the shell of the php container
```bash
wodby@php.container:/var/www/html $ ./vendor/bin/drush php
Psy Shell v0.11.9 (PHP 8.1.13 — cli) by Justin Hileman
FROMATION (Drupal 9.4.8)
> \Drupal::state()->set('my_unique_keyname', 'value')
= null
> \Drupal::state()->get('my_unique_keyname')
= "value"
```
* so passing the database request on [phpMyAdmin](http://pma.drupal.docker.localhost:8000) gives the value result inside the BLOB:
  * the table and collection name are to be found on p 142
```sql
SELECT * FROM `key_value` where collection = 'state' and name='my_unique_keyname';
```
* inside the BLOB we have _s:5:"value";_

### the KeyValueDatabaseFactory as a service

* in the _docker4drupal/web/core/core.services.yml_ at line 475 you have the State Description as a Drupal service
```yaml
state:
    class: Drupal\Core\State\State
    arguments: ['@keyvalue']
```
* in the State constructor we have:
```php
  /**
   * Constructs a State object.
   *
   * @param \Drupal\Core\KeyValueStore\KeyValueFactoryInterface $key_value_factory
   *   The key value store to use.
   */
  public function __construct(KeyValueFactoryInterface $key_value_factory) {
    $this->keyValueStore = $key_value_factory->get('state');
  }
```
* in the same file at line 419:
```
  keyvalue:
    class: Drupal\Core\KeyValueStore\KeyValueFactory
    arguments: ['@service_container', '%factory.keyvalue%']
  keyvalue.database:
    class: Drupal\Core\KeyValueStore\KeyValueDatabaseFactory
    arguments: ['@serialization.phpserialize', '@database']
```
* What is the link betwee _keyvalue_ and _keyvalue.database_ ?
* the service definition yaml file comes from [Symfony links which explains the percent sign](https://symfonycasts.com/screencast/drupal8-under-the-hood/config-parameters)
  * when you use %% it is not an existing service but an existing parameter (in Symfony inside the _parameter.yml_ file) which is replaced by its value!
* inside the _parameters_ section of the core.yml we have
```yaml
  factory.keyvalue:
    default: keyvalue.database
``` 
* Drupal's _Drupal\Core\KeyValueStore\KeyValueFactory_ getter is
  * static::DEFAULT_SETTING is default
  * at the beginning state collection is not known from the stores nor from the options
  * so we are using _$service_id = $this->options[static::DEFAULT_SETTING];_ wihich calls then _keyvalue.database_ service.
```php
/**
   * {@inheritdoc}
   */
  public function get($collection) {
    if (!isset($this->stores[$collection])) {
      if (isset($this->options[$collection])) {
        $service_id = $this->options[$collection];
      }
      elseif (isset($this->options[static::DEFAULT_SETTING])) {
        $service_id = $this->options[static::DEFAULT_SETTING];
      }
      else {
        $service_id = static::DEFAULT_SERVICE;
      }
      $this->stores[$collection] = $this->container->get($service_id)->get($collection);
    }
    return $this->stores[$collection];
  }
```
* the _$this->container->get($service_id)_ returns the _KeyValueDatabaseFactory_
  * which from the State object returns a _new DatabaseStorage('state', $this->serializer, $this->connection)_ 
    * in fact the _Drupal\Core\KeyValueStore\KeyValueDatabaseFactory_ has also a get method which takes a collection like _states_ in our example:
```php
  /**
   * {@inheritdoc}
   */
  public function get($collection) {
    return new DatabaseStorage($collection, $this->serializer, $this->connection);
  }
```
* _DatabaseStorage_ is in the same folder and it calls queries by default on the __key_value__ table:
```php
  /**
   * Overrides Drupal\Core\KeyValueStore\StorageBase::__construct().
   *
   * @param string $collection
   *   The name of the collection holding key and value pairs.
   * @param \Drupal\Component\Serialization\SerializationInterface $serializer
   *   The serialization class to use.
   * @param \Drupal\Core\Database\Connection $connection
   *   The database connection to use.
   * @param string $table
   *   The name of the SQL table to use, defaults to key_value.
   */
  public function __construct($collection, SerializationInterface $serializer, Connection $connection, $table = 'key_value') {
    parent::__construct($collection); //in ou State case, collection = 'state'
    $this->serializer = $serializer;
    $this->connection = $connection;
    $this->table = $table;
  }

  //and later
    /**
   * {@inheritdoc}
   */
  public function getMultiple(array $keys) {
    $values = [];
    try {
      $result = $this->connection->query('SELECT [name], [value] FROM {' . $this->connection->escapeTable($this->table) . '} WHERE [name] IN ( :keys[] ) AND [collection] = :collection', [':keys[]' => $keys, ':collection' => $this->collection])->fetchAllAssoc('name');
      foreach ($keys as $key) {
        if (isset($result[$key])) {
          $values[$key] = $this->serializer->decode($result[$key]->value);
        }
      }
    }
    catch (\Exception $e) {
      // @todo: Perhaps if the database is never going to be available,
      // key/value requests should return FALSE in order to allow exception
      // handling to occur but for now, keep it an array, always.
    }
    return $values;
  }

```
* the State object  calls its _$loaded_values = $this->keyValueStore->getMultiple($load);_

# The tempstore:

## The private Tempstore

* The _PrivateTempStore_ is an API for dealing with personal temporary data
  * it is not a service but is created by a service (see line 1786 in _docker4drupal/web/core/core.services.yml_)

```yaml
tempstore.private:
    class: Drupal\Core\TempStore\PrivateTempStoreFactory
    arguments: ['@keyvalue.expirable', '@lock', '@current_user', '@request_stack', '%tempstore.expire%']
    tags:
      - { name: backend_overridable }
```

### more about tags in the service yaml file

* [Services Tags](https://symfony.com/doc/current/service_container/tags.html) come from Symfony
* Especially the chapter [Creating custom tags](https://symfony.com/doc/current/service_container/tags.html#creating-custom-tags)
  * is worth reading
  * we define a tag to have a class that collect all services that have the tag in question
  * like _MailTransportPass_ which implements _CompilerPassInterface_ and is registered in the Kernel.php (like a bundle)

```php
// find all service IDs with the app.mail_transport tag
$taggedServices = $container->findTaggedServiceIds('app.mail_transport');

foreach ($taggedServices as $id => $tags) {
    // add the transport service to the TransportChain service
    $definition->addMethodCall('addTransport', [new Reference($id)]);
}
```

### back to the private TempStore:

* We obtain a _PrivateTempStore_ from the __docker4drupal/web/core/lib/Drupal/Core/TempStore/PrivateTempStoreFactory.php__ through its _get_ method:
```php
   /**
   * Creates a PrivateTempStore.
   *
   * @param string $collection
   *   The collection name to use for this key/value store. This is typically
   *   a shared namespace or module name, e.g. 'views', 'entity', etc.
   *
   * @return \Drupal\Core\TempStore\PrivateTempStore
   *   An instance of the key/value store.
   */
  public function get($collection) {
    // Store the data for this collection in the database.
    $storage = $this->storageFactory->get("tempstore.private.$collection");
    return new PrivateTempStore($storage, $this->lockBackend, $this->currentUser, $this->requestStack, $this->expire);
  }
```

* The StorageFactroy is the first parameter of the _PrivateTempStoreFactory_ in the core.yml file:
* Which bring us to:
* line 425 of core.yml
```yaml
keyvalue.expirable:
    class: Drupal\Core\KeyValueStore\KeyValueExpirableFactory
    arguments: ['@service_container', '%factory.keyvalue.expirable%']
keyvalue.expirable.database:
  class: Drupal\Core\KeyValueStore\KeyValueDatabaseExpirableFactory
  arguments: ['@serialization.phpserialize', '@database']
```
* with the parameter _factory.keyvalue.expirable_ (line 33 of the same core.yaml)
```yaml
factory.keyvalue.expirable:
    default: keyvalue.expirable.database
```
#### KeyValueExpirableFactory

* like _docker4drupal/web/core/lib/Drupal/Core/KeyValueStore/KeyValueDatabaseFactory.php_ it is in
* _docker4drupal/web/core/lib/Drupal/Core/KeyValueStore/KeyValueExpirableFactory.php_
* it only define 3 constants and extends _docker4drupal/web/core/lib/Drupal/Core/KeyValueStore/KeyValueFactory.php_
* The storage I return for _PrivateTempStore_ id done in the get($option) of _PrivateTempStoreFactory_ 
* which calls the service of id __keyvalue.expirable.database__ in the get method of _KeyValueFactory_:
  * from _PrivateTempStoreCollection_, the collection is in fact _"tempstore.private.$collection"_ 
```php
 /**
   * {@inheritdoc}
   */
  public function get($collection) {
    if (!isset($this->stores[$collection])) {
      if (isset($this->options[$collection])) {
        $service_id = $this->options[$collection];
      }
      elseif (isset($this->options[static::DEFAULT_SETTING])) {
        $service_id = $this->options[static::DEFAULT_SETTING];
      }
      else {
        $service_id = static::DEFAULT_SERVICE;
      }
      $this->stores[$collection] = $this->container->get($service_id)->get($collection);
    }
    return $this->stores[$collection];
  }
```
#### Drupal\Core\KeyValueStore\KeyValueDatabaseExpirableFactory

* see line 428 of _docker4drupal/web/core/core.services.yml_
```yaml
keyvalue.expirable.database:
    class: Drupal\Core\KeyValueStore\KeyValueDatabaseExpirableFactory
    arguments: ['@serialization.phpserialize', '@database']
```
* it has a get service which returns a _DatabaseStorageExpirable_ object
```php
/**
   * {@inheritdoc}
   */
  public function get($collection) {
    if (!isset($this->storages[$collection])) {
      $this->storages[$collection] = new DatabaseStorageExpirable($collection, $this->serializer, $this->connection);
    }
    return $this->storages[$collection];
  }
```
* this _docker4drupal/web/core/lib/Drupal/Core/KeyValueStore/DatabaseStorageExpirable.php_ 
  * works with the _key_value_expire_
  * returns a value only if the field expire: _WHERE [expire] > :now_
```php
/**
 * Defines a default key/value store implementation for expiring items.
 *
 * This key/value store implementation uses the database to store key/value
 * data with an expire date.
 */
class DatabaseStorageExpirable extends DatabaseStorage implements KeyValueStoreExpirableInterface {

  /**
   * Overrides Drupal\Core\KeyValueStore\StorageBase::__construct().
   *
   * @param string $collection
   *   The name of the collection holding key and value pairs.
   * @param \Drupal\Component\Serialization\SerializationInterface $serializer
   *   The serialization class to use.
   * @param \Drupal\Core\Database\Connection $connection
   *   The database connection to use.
   * @param string $table
   *   The name of the SQL table to use, defaults to key_value_expire.
   */
  public function __construct($collection, SerializationInterface $serializer, Connection $connection, $table = 'key_value_expire') {
    parent::__construct($collection, $serializer, $connection, $table);
  }
```
* and for the _database_ service see line 382 of _docker4drupal/web/core/core.services.yml_
  * it is the same as used by the _State_ service
```yaml
 database:
    class: Drupal\Core\Database\Connection
    factory: Drupal\Core\Database\Database::getConnection
    arguments: [default]
```
#### Conclusion for the PrivateTempStore:
* Table: _key_value_expire_
* collection _tempstore.private.$collection_
```sql
select * from key_value_expire;  --does not return anything
```
## The Shared TempStore
 
* In the same way, the class _docker4drupal/web/core/lib/Drupal/Core/TempStore/SharedTempStoreFactory.php_
* generates a _docker4drupal/web/core/lib/Drupal/Core/TempStore/SharedTempStore.php_
* in the constructor we have a _trigger_error_ which writes a DEPRECATED message without stopping the flow of execution:
```php
@trigger_error('Calling ' . __METHOD__ . '() without the $current_user argument is deprecated in drupal:9.2.0 and will be required in drupal:10.0.0. See https://www.drupal.org/node/3006268', E_USER_DEPRECATED);
```
### Notes about the SharedTempStore:
* It looks for locking if the key exsits before atempting to update its value!
* setting the value means setting the expirable value apart from the rest of the object:
```php
  /**
   * Stores a particular key/value pair in this SharedTempStore.
   *
   * @param string $key
   *   The key of the data to store.
   * @param mixed $value
   *   The data to store.
   *
   * @throws \Drupal\Core\TempStore\TempStoreException
   *   Thrown when a lock for the backend storage could not be acquired.
   */
  public function set($key, $value) {
    if (!$this->lockBackend->acquire($key)) {
      $this->lockBackend->wait($key);
      if (!$this->lockBackend->acquire($key)) {
        throw new TempStoreException("Couldn't acquire lock to update item '$key' in '{$this->storage->getCollectionName()}' temporary storage.");
      }
    }

    $value = (object) [
      'owner' => $this->owner,
      'data' => $value,
      'updated' => (int) $this->requestStack->getMainRequest()->server->get('REQUEST_TIME'),
    ];
    $this->ensureAnonymousSession();
    $this->storage->setWithExpire($key, $value, $this->expire);
    $this->lockBackend->release($key);
  }
```
#### The Storage setWithExpire function

* to be found at: _docker4drupal/web/core/lib/Drupal/Core/KeyValueStore/DatabaseStorageExpirable.php_
  * in the constructor: __this->table=key_value_expire__
  * the collection is __"tempstore.private.$collection"__
```php
  protected function doSetWithExpire($key, $value, $expire) {
    $this->connection->merge($this->table)
      ->keys([
        'name' => $key,
        'collection' => $this->collection,
      ])
      ->fields([
        'value' => $this->serializer->encode($value),
        'expire' => REQUEST_TIME + $expire,
      ])
      ->execute();
  }
```
* back to the merge method of the Connection class
  * reminder:
```yaml
 database:
    class: Drupal\Core\Database\Connection
    factory: Drupal\Core\Database\Database::getConnection
    arguments: [default]
```
#### Merge.php class
* which sends us to the docker4drupal/web/core/lib/Drupal/Core/Database/Query/Merge.php class:
  * which is an abstract class:
* keys creates the conditions
```php
$this->conditions[] = [
      'field' => $field,
      'value' => $value,
      'operator' => $operator, //operator by default is =
    ];
```
* _fields_ function of the Merge class:
```php 
foreach ($fields as $key => $value) {
  $this->insertFields[$key] = $value;
  $this->updateFields[$key] = $value;
}
```
* everything is in the _execute_ function:
  * from Line 371
  * it inserts or updates depending of a condition table which by default is my table