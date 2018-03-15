# Crypt Field - Encrypt field values at the field sql storage level

This is a module for Drupal 7. This module is an alternate field storage engine,
using SQL the same way the core field does.

**THIS IS AN EXPERIMENTAL PROJECT AND HAS NOT BEEN AUDITED BY SECURITY EXPERTS,**
**YET,**
**USE AT YOUR OWN RISKS!**

# Getting started

Download and install this module using composer:

```sh
composer require makinacorpus/drupal-cryptfield
```

If you can't use composer, you must also install the ``paragonie/sodium_compat``
PHP package manually if you are using PHP < 7.2. You can find the source code
and documentation there: https://github.com/paragonie/sodium_compat

Then enable the module:

```sh
drush en cryptfield
```

For using the storage engine instead of Drupal core's ``field_storage_sql`` own
engine, refer to Drupal core documentation. If you are creating your fields
manually during hook_install() or hook_update_N() execution, you can add
the following data within your field description array:

```php
function mymodule_install() {
  field_create_field([
    'field_name' => 'my_first_encrypted_field',
    'type' => 'text_long_with_summary',
    // Append this:
    'storage' => [
      'type' => 'cryptfield',
    ],
  ]);
}
```

As of today, the engine does not support any options at the field level yet
(it probably will soon).

# Important limitations

**You cannot use entity field queries as known as ``EFQ`` with this engine.**
**It makes no sense to index encrypted data: it would leak the data iself**
**by reading or attacking the indexes.**

# Technical overview

We are not cryptography experts, if anything in this is wrong, reviews will
be gladly welcomed, patches too.

## Strong and modern encryption

As stated upper, we are not cryptography experts, and we do trust people who
are, [Paragonie](https://paragonie.com), which did ported ``libsodium`` to PHP
are. If you are using PHP 7.2, libsodium is included into PHP core itself.

## Type of encryption in use

Basic secret-key encryption using ``sodium_secretbox_*`` is being used, we have
no use in using asymetric private/public key encryption, not until we deal with
user's own need for privacy.

Read more about this topic
[on Paragonie's documentation](https://paragonie.com/book/pecl-libsodium/read/04-secretkey-crypto.md)
site.

## How are stored the keys

A single site-wide private key will be used to encrypt the field values,
nevertheless, to enfore maximum security, each field row in database will
carry its own ``nonce`` (sometime called ``salt`` or ``iv``), then:

 * The shared private key itself is stored into a file on disk, per default
   it is being stored in ``private://cryptfield.key``.

 * For maximum security, this key is itself encrypted using another key stored
   in configuration under the ``cryptfield_secretbox_key_key`` variable:
   **when deploying a site in production, you should set this variable into**
   **your ``settings.php`` file**.

 * The nonce used to decrypt the private key itself is stored in the variables
   as well, under the ``cryptfield_configuration_nonce`` variable.

If configured correctly (i.e. with the ``cryptfield_secretbox_key_key`` variable
into the ``settings.php``, this means that, in order to steal your private key,
an attacker would need to do all of:

 1. be able to read the settings.php contents,
 2. be able to read your private filesystem (which may be hosted under a
    different filesystem, with different permissions),
 3. be able to read within your database or cache backend (where the nonce
    lies).

Only then, the attacker could decrypt the private key.

Please note I'm not really sure this is useful in the end, but it does work.

# Maybe future plans

 * Add an option at the field level that allows only a configured user to
   decrypt field's content: for this, a per-user private key needs to be
   stored along the user structure.

 * Create one single table for all fields on the same entity that have a
   cardinality set to 1 (yes, we do have serious performance problems on
   some sites).
