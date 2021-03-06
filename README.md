# Prestashop ObjectModel Extension

1. [About](https://github.com/creativewave/ps-objectmodel-extension#about)
2. [Installation](https://github.com/creativewave/ps-objectmodel-extension#installation)
3. [Usage](https://github.com/creativewave/ps-objectmodel-extension#usage)
4. [How it works](https://github.com/creativewave/ps-objectmodel-extension#how-it-works)

## About

This library provides **a simple and declarative way to create/drop database tables** accessed from models (classes extending `ObjectModel`) used by Prestashop (1.6) modules.

It makes use of `<MyModel>::$definition` to create the corresponding table name, columns, keys and indexes, as well as for each relations tables: multilang/shop, or any "OneToMany/ManyToMany" relation (aka `'associations'`).

Each table will be created with the expected keys. Eg.: `ps_my_model_shop` will have a composite primary key combined from columns `id_my_model` and `id_shop`, and foreign keys referencing `ps_my_model.id_my_model` and `ps_shop.id_shop`.

Each column will be created with the expected type and constraints. Eg.: a string field type defined with `'size' => 40` will be translated to `VARCHAR(40)`. Fields could even be defined to be used as a unique, fulltext, or simple key.

Note: unlike multilang/shop fields, relations fields can't be handled by `ObjectModel` when inserting a new model entry in database. It must be done manually (as well as for validating multishop fields).

## Installation

This library should be installed with Composer. Edit your `composer.json` file:

```json
"repositories": [
  {
    "type": "git",
    "url": "https://github.com/creativewave/ps-objectmodel-extension"
  }
],
"require": {
  "creativewave/ps-objectmodel-extension": "^1"
}
```

Run `composer install`.

## Usage

Include the autoloader generated by Composer at the top of `<MyModule>`, or in its install/uninstall methods:

```php
<?php

require_once _PS_ROOT_DIR_.'/vendor/autoload.php';

class MyModule extends Module
{
    //...
```

In `<MyModule>::install()`, create an instance of `CW\ObjectModel\Extension` by passing it instances of `<MyModel>` and `Db`. It will return an ObjectModel extended (decorated) with `install` and `uninstall` methods.

```php
public function install(): bool
{
    $model = new MyModel();
    $db = Db::getInstance();
    $extended_model = new CW\ObjectModel\Extension($model, $db);

    return parent::install() and $extended_model->install();
}

public function uninstall(): bool
{
    $model = new MyModel();
    $db = Db::getInstance();
    $extended_model = new CW\ObjectModel\Extension($model, $db);

    return parent::uninstall() and $extended_model->uninstall();
}
```

Finally, define model, fields, and associations in `<MyModel>::$definition`.

Supported fields types parameters (`ObjectModel::TYPE`s) and their correspondances to an SQL type:

- [x] BOOL > TINYINT(1)
- [x] DATE > DATETIME
- [x] FLOAT > DECIMAL (use `'size' => <precision>, 'scale' => <scale> ]`)
- [x] HTML > TEXT
- [x] INT > INT
- [x] SQL > TEXT
- [x] STRING > VARCHAR (size, aka length, will default to 255)
- [ ] NOTHING

All other fields parameters are supported and are logically translated (lang, shop, default, allow_null, required, values, size, validate).

### Optional: registering simple/unique/fulltext keys

Keys (indexes) can be created in order to improve SQL requests performances:

```php
protected static $definition = [
    'fields' => [
        'my_unique_field' => [
            'type'     => ObjectModel::TYPE_STRING,
            'validate' => 'isGenericName',
            'key'      => true,
            // Or 'unique'   => true
            // Or 'fulltext' => true
        ],
    ];
];
```

See [Rick James rules of thumbs](http://mysql.rjweb.org/doc.php/ricksrots) to learn more about when/how to use indexes.

### Optional: qualifying a relation

You can qualify a ManyToMany relation (aka multi`<relation>`) table with additional fields/columns.

Best way to see how it could be usefull is to imagine a product/order relation. Orders could have many products and products could be in many orders: they need an intermediary table, containing columns `id_product` and `id_order`. But you may also want to specify a product order quantity, with a column named `product_quantity`: an extra column qualifying the relation.

To register your qualifying field, define it within a `fields` array in `<MyModel>::$definition['associations']`:

```php
protected static $definition = [
    'associations' => [
        'products' => [
            'type'        => ObjectModel::HAS_MANY,
            'object'      => 'Product',    // Relation model class name.
            'association' => 'product',    // Relation table name.
            'field'       => 'id_product', // Relation table primary key.
            'fields'      => [
                'product_quantity' => [
                    'type' => //...
                ],
            ],
        ],
    ];
];
```

Finally, you can set this intermediary table as a multilang/shop table, by adding the corresponding key/value (`boolean`) pair:

```php
protected static $definition = [
    'associations' => [
        'products' => [
            'type'        => ObjectModel::HAS_MANY,
            'object'      => 'Product',
            'association' => 'product',
            'field'       => 'id_product',
            'multishop'   => true,
        ],
    ];
];
```

Columns `id_shop` and/or `id_lang` will be automatically created, with foreign key(s) referencing `ps_shop` and/or `ps_lang`.

## How it works

There is basically 3 main components:

* ObjectModel\Extension: the runtime
* ObjectModel\Definition: an abstraction of `<MyObjectModel::$definition>`
* Db\Table: an abstraction of a table in the database

More deeply, it also uses:

* Db\Schema: a "protocol" to get and set Db\Table properties
* Db\Table\Definition\Model: an abstraction of the model table
* Db\Table\Definition\Relation: an abstraction of a relation table

## Todo

* Feature: updating tables schemas (when updating modules)
* Feature: handle `size` field parameter for setting a numeric type and its length
* Feature: creating custom composite keys
* Feature: creating custom prefix keys
* Feature: creating custom keys in relation tables
