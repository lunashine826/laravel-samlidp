# Laravel SAML idP

This package allows you to implement your own Identification Provider (idP) using the SAML 2.0 standard to be used with supporting SAML 2.0 Service Providers (SP).

## Installation

Require this package with composer:

```shell
composer require codegreencreative/laravel-samlidp
```

Laravel 5.6+ should auto discover the package, if not, run

```shell
php artisan package:discover
```

Publish config
config/samlidp.php

```shell
php artisan vendor:publish --tag="samlidp_config"
```

FileSystem configuration
Within `config/filesystem.php` following entry needs to be added:

```php
'disks' => [

        ...

        'samlidp' => [
            'driver' => 'local',
            'root' => storage_path() . '/samlidp',
        ]
],
```

# Create a Self Signed Certificate (to be used later)

Use the following command to create new certificate and private key for yoru IdP.

```shell
php artisan samlidp:cert --days 7300 --keyname key --certname cert
```

--days <int>
>Number of days to add from today as the expiration date
>Default: 7300

--keyname <string>
>Prefix name to the key file
>Default: key.pem

--certname <string>
>Prefix name to the certificate file
>Default: cert.pem

## Usage

Within your login view, problably `resources/views/auth/login.blade.php` add the SAMLRequest directive beneath the CSRF directive:

```php
@csrf
@samlidp
```

The SAMLRequest directive will fill out the hidden input automatically when a SAMLRequest is sent by an HTTP request and therefore initiate a SAML authentication attempt. To initiate the SAML auth, the login and redirect processes need to be intervened. This is done using the Laravel events fired upon authentication.

## Config

After you publish the config file, you will need to set up your Service Providers. The key for the Service Provider is a base 64 encoded Consumer Service (ACS) URL. You can get this information from your Service Provider, but you will need to base 64 encode the URL and place it in your config. This is due to config dot notation.

You may use this command to help generate a new SAML Service Provider:

```shell
php artisan samlidp:sp
```

Example SP in `config/samlidp.php` file:

```php
<?php

return [
    // The URI to your login page
    'login_uri' => 'login',
    // The URI to the saml metadata file, this describes your idP
    'issuer_uri' => 'saml/metadata',
    // List of all Service Providers
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ACS URL of the Service Provider
            'destination' => 'https://example.com/saml/acs',
            // Simple Logout URL of the Service Provider
            'logout' => 'https://example.com/saml/sls',
        ]
    ]

];
```

## Attributes (optional)

Service providers may require more additional attributes to be sent via assertion. Its even possible that they require the same information but as a different Claim Type.

By Default this package will send the following Claim Types:

`ClaimTypes::EMAIL_ADDRESS` as `auth()->user()->email`
`ClaimTypes::GIVEN_NAME` as `auth()->user()->name`

This is because Laravel migrations, by default, only supply email and name fields that are usable by SAML 2.0.

To add additional Claim Types, you can subscribe to the Assertion event:

`CodeGreenCreative\SamlIdp\Events\Assertion`

In 

Sample Listener:

```php
<?php

namespace App\Listeners;

use LightSaml\ClaimTypes;
use LightSaml\Model\Assertion\Attribute;
use CodeGreenCreative\SamlIdp\Events\Assertion;

class SamlAssertionAttributes
{
    public function handle(Assertion $event)
    {
        $event->attribute_statement
            ->addAttribute(new Attribute(ClaimTypes::PPID, auth()->user()->id))
            ->addAttribute(new Attribute(ClaimTypes::NAME, auth()->user()->name));
    }
}

```