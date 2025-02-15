---
description: To be able to store the data saved to a Field, you must configure storage conversion for the Field Type.
---

# Field Type storage

## Storage conversion

If you want to store Field values in regular [[= product_name =]] database tables,
the `FieldValue` must be converted to the storage-specific format used by the Persistence SPI:
`Ibexa\Contracts\Core\Persistence\Content\FieldValue`.
After restoring a Field of the Field Type, you must reverse the conversion.

The following methods of the Field Type are responsible for that:

|Method|Description|
|------|-----------|
|`toPersistenceValue()`|This method receives the value of a Field of the Field Type and returns an SPI `FieldValue`, which can be stored.|
|`fromPersistenceValue()`|This method receives an SPI `FieldValue` and reconstructs the original value of the Field from it.|

The SPI `FieldValue` struct has properties which the Field Type can use:

|Property|Description|
|--------|-----------|
|`$data`|The data to be stored in the database. This may be a scalar value, an associative array or a simple, serializable object.|
|`$externalData`|The arbitrary data stored in this field will not be touched by any of the [[= product_name =]] components directly, but will be available for [Storing external data](#storing-external-data).|
|`$sortKey`|A value which can be used to sort content by this Field.|

### Legacy storage engine

The Legacy storage engine uses the `ezcontentobject_attribute` table to store Field values,
and `ezcontentclass_attribute` to store Field definition values.
They are both based on the same principle.

Each row represents a Field or a Field definition, and offers several free fields of different types, where the type can store its data e.g.

- `ezcontentobject_attribute` offers:
    - `data_int`
    - `data_text`
    - `data_float`
- `ezcontentclass_attribute` offers:
    - four `data_int` (`data_int1` to `data_int4`) fields
    - four `data_float` (`data_float1` to `data_float4`) ones
    - five `data_text` (`data_text1` to `data_text5`)

Each type is free to use those fields in any way it requires.

The default Legacy storage engine cannot store arbitrary value information as provided by a Field Type.
This means that using this storage engine requires a conversion.
Converters will map a Field's semantic values to the fields described above, for both settings (validation and configuration) and value.

The conversion takes place through the `Ibexa\Core\Persistence\Legacy\Content\FieldValue\Converter` interface,
which you must implement in your Field Type. The interface contains the following methods:

|Method|Description|
|------|-----------|
|`toStorageValue()`|Converts a Persistence `Value` into a Legacy storage specific value.|
|`toFieldValue()`|Converts the other way around.|
|`toStorageFieldDefinition()`|Converts a Persistence `FieldDefinition` to a storage specific one.|
|`toFieldDefinition`|Converts the other way around.|
|`getIndexColumn()`|Returns the storage column which is used for indexing either `sort_key_string` or `sort_key_int`.|

Just like a Type, a Legacy Converter needs to be registered and tagged in the [service container](php_api.md#service-container).

#### Registering a converter

The registration of a `Converter` currently works through the `$config` parameter of [`Ibexa\Core\Persistence\Legacy\Handler`](https://github.com/ibexa/core/blob/main/src/lib/Persistence/Legacy/Handler.php).

Those converters also need to be correctly exposed as services and tagged with `ibexa.field_type.storage.legacy.converter`:

``` yaml
services:
    Ibexa\Core\Persistence\Legacy\Content\FieldValue\Converter\TextLine:
        tags:
            - {name: ibexa.field_type.storage.legacy.converter, alias: ezstring}
```

The tag has the following attribute:

| Attribute name | Usage |
|----------------|-------|
| `alias` | Represents the `fieldTypeIdentifier` (just like for the [Field Type service](type_and_value.md#registration)). |

!!! tip

    Converter configuration for built-in Field Types is located in [`ibexa/core/src/lib/Resources/settings/fieldtype_external_storages.yml`](https://github.com/ibexa/core/blob/main/src/lib/Resources/settings/fieldtype_external_storages.yml).

## Storing data externally

A Field Type may store arbitrary data in external data sources.
External storage can be e.g. a web service, a file in the file system, another database
or even the [[= product_name =]] database itself (in form of a non-standard table).

In order to store data in external storage, the Field Type will interact with the Persistence SPI
through the `Ibexa\Contracts\Core\FieldType\FieldStorage` interface.

Accessing the internal storage of a Content item that includes a Field of the Field Type
calls one of the following methods to also access the external data:

|Method|Description|
|------|-----------|
|`hasFieldData()`|Returns whether the Field Type stores external data at all.|
|`storeFieldData()`|Called right before a Field of the Field Type is stored. The method stores `$externalData`. It returns `true` if the call manipulated internal data of the given Field, so that it is updated in the internal database.|
|`getFieldData()`|Called after a Field has been restored from the database in order to restore `$externalData`.|
|`deleteFieldData()`|Must delete external data for the given Field, if exists.|
|`getIndexData()`|Returns the actual index data for the provided `Ibexa\Contracts\Core\Persistence\Content\Field`. For more information, see [search service](field_type_search.md#search-field-values).|

Each of the above methods (except `hasFieldData`) receives a `$context` array with information on the underlying storage and the environment.
To retrieve and store data in the [[= product_name =]] data storage,
but outside of the normal structures (e.g. a custom table in an SQL database),
use [Gateway-based storage](#gateway-based-storage) with properly injected Doctrine Connection.

Note that the Field Type must take care on its own for being compliant with different data sources and that third parties can extend the data source support easily.

### Gateway-based storage

In order to allow the usage of a Field Type that uses external data with different data storages, it is recommended to implement a gateway infrastructure and a registry for the gateways. To make this easier, the Core implementation of Field Types provides corresponding interfaces and base classes. They can also be used for custom Field Types.

The interface `Ibexa\Contracts\Core\FieldType\StorageGateway` is implemented by gateways, in order to be handled correctly by the registry. It has one method:

|Method|Description|
|------|-----------|
|`setConnection()`|The registry mechanism uses this method to set the SPI storage connection (e.g. the database connection to the Legacy Storage database) into the gateway, which might be used to store external data. The connection is retrieved from the `$context` array automatically by the registry.|

Note that the Gateway implementation itself must take care of validating that it received a usable connection. If it does not, it should throw a `RuntimeException`.

The registry mechanism is realized as a base class for `FieldStorage` implementations: `Ibexa\Core\FieldType\GatewayBasedStorage`. For managing `StorageGateway`s, the following methods are already implemented in the base class:

|Method|Description|
|------|-----------|
|`addGateway()`|Allows the registration of additional `StorageGateway`s from the outside. Furthermore, an associative array of `StorageGateway`s can be given to the constructor for basic initialization. This array should originate from the dependency injection mechanism.|
|`getGateway()`|This protected method is used by the implementation to retrieve the correct `StorageGateway` for the current context.|

!!! tip

    Refer to the built-in Keyword, URL and User Field Types for usages of such infrastructure.

### Registering external storage

To use external storage, you need to define a service implementing the `Ibexa\Contracts\Core\FieldType\FieldStorage` interface
and tag it as `ibexa.field_type.storage.external.handler` to be recognized by the Repository.

Here is an example for the `myfield` Field Type:

``` yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        public: false

    App\FieldType\MyField\Storage\MyFieldStorage: ~
        tags:
            - {name: ibexa.field_type.storage.external.handler, alias: myfield}
```

The configuration requires providing the `ibexa.field_type.storage.external.handler` tag, with the `alias` attribute being the *fieldTypeIdentifier*. You also have to inject the gateway in `arguments`, [see Gateway-based storage](#gateway-based-storage).

External storage configuration for basic Field Types is located in [`ibexa/core/src/lib/Resources/settings/fieldtype_external_storages.yml`](https://github.com/ibexa/core/blob/main/src/lib/Resources/settings/fieldtype_external_storages.yml).

Using gateway-based storage requires another service implementing `Ibexa\Core\FieldType\StorageGateway` to be injected into the [external storage handler](#storing-external-data)).

``` yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        public: false

    App\FieldType\MyField\Storage\Gateway\DoctrineStorage: ~
```

Note that `ibexa.api.storage_engine.legacy.connection` is of type `Doctrine\DBAL\Connection`. If your gateway still uses an implementation of `eZ\Publish\Core\Persistence\Database\DatabaseHandler` (`eZ\Publish\Core\Persistence\Doctrine\ConnectionHandler`), instead of the `ibexa.api.storage_engine.legacy.connection`, you can pass the `ibexa.api.storage_engine.legacy.dbhandler` service.


Also note that there can be several gateways per Field Type (one per storage engine). In this case it's recommended to either create base implementation which each gateway can inherit or create interface which each gateway must implement and reference it instead of specific implementation when type-hinting method arguments.

!!! tip

    Gateway configuration for built-in Field Types is located in [`core/src/lib/Resources/settings/storage_engines/`](https://github.com/ibexa/core/tree/main/src/lib/Resources/settings/storage_engines).

## Storing Field Type settings externally

Just like in the case of data, storing [Field Type settings](type_and_value.md#field-type-settings) 
in Content Item tables may prove insufficient. 
It is not a problem if your setting specifies, for example, just the allowed number of characters 
in a text field. 
However, the Field Type may represent a more complex object, for example, it may 
consist of two or more other fields, such as the name, SKU and price, and there 
can be a set of default values instead of just one. 
Once you add validation rules for these field values, then it becomes an issue.
    
You can overcome this obstacle: 
When you create a new Field Type, you can move Field Type settings 
to external storage.
    
!!! note
    
    Another benefit of an external storage is that there can be database relations to 
    other objects/entities, and the database itself can maintain the integrity of data.
    
First, create a class that implements the 
`Ibexa\Contracts\Core\FieldType\FieldConstraintsStorage` interface.
    
Then, register the External Storage as a service and tag it with `ibexa.field_type.external_constraints_storage`. 
Make sure that the alias you use matches the identifier of the new Field Type:
    
``` yaml
services:
    App\FieldType\Example\ExternalStorage:
        tags:
            - { name: ibexa.field_type.external_constraints_storage, alias: <field_type_identifier> }
```
