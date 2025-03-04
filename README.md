# letsencrypt [![CircleCI](https://circleci.com/gh/publishlab/node-acme-client.svg?style=svg)](https://circleci.com/gh/publishlab/node-acme-client)

*A simple and unopinionated Collection of Modules Implementing the LetsEncrypt API's.*

* RFC 8555 - Automatic Certificate Management Environment (ACME): [https://tools.ietf.org/html/rfc8555](https://tools.ietf.org/html/rfc8555)
* Boulder divergences from ACME: [https://github.com/letsencrypt/boulder/blob/master/docs/acme-divergences.md](https://github.com/letsencrypt/boulder/blob/master/docs/acme-divergences.md)

## Important upgrade notice

On September 15, 2022, Let's Encrypt will stop accepting Certificate Signing Requests signed using the obsolete SHA-1 hash. This change affects all `acme-client` versions lower than `3.3.2` and `4.2.4`.

A more detailed explanation can be found [at the Let's Encrypt forums](https://community.letsencrypt.org/t/rejecting-sha-1-csrs-and-validation-using-tls-1-0-1-1-urls/175144).

### Compatibility

| letsencrypt/acme-client   | Node.js   |                                           |
| ------------- | --------- | ----------------------------------------- |
| v6.x          | >= v19    | [Upgrade guide](docs/wip-v6.md)       |
| v5.x          | >= v16    | [Upgrade guide](docs/upgrade-v5.md)       |
| v4.x          | >= v10    | [Changelog](CHANGELOG.md#v400-2020-05-29) |
| v3.x          | >= v8     | [Changelog](CHANGELOG.md#v300-2019-07-13) |
| v2.x          | >= v4     | [Changelog](CHANGELOG.md#v200-2018-04-02) |
| v1.x          | >= v4     | [Changelog](CHANGELOG.md#v100-2017-10-20) |


### Table of contents

* [Installation](#installation)
* [Usage](#usage)
    * [Directory URLs](#directory-urls)
    * [External account binding](#external-account-binding)
    * [Specifying the account URL](#specifying-the-account-url)
* [Cryptography](#cryptography)
    * [Legacy .forge interface](#legacy-forge-interface)
* [Auto mode](#auto-mode)
    * [Challenge priority](#challenge-priority)
    * [Internal challenge verification](#internal-challenge-verification)
* [API](#api)
* [HTTP client defaults](#http-client-defaults)
* [Debugging](#debugging)
* [License](#license)




## Installation & Usage 

```js
import letsencrypt from './letsencrypt.js';

const accountPrivateKey = '<PEM encoded private key>';

const client = new letsencrypt.Client({
    directoryUrl: acme.directory.letsencrypt.staging,
    accountKey: accountPrivateKey
});
```


### Directory URLs

```js
acme.directory.bypass.staging;
acme.directory.bypass.production;

acme.directory.letsencrypt.staging;
acme.directory.letsencrypt.production;

acme.directory.zerossl.production;
```


### External account binding

To enable [external account binding](https://tools.ietf.org/html/rfc8555#section-7.3.4) when creating your ACME account, 
provide your KID and HMAC key to the client constructor.

```js
const client = new letsencrypt.Client({
    directoryUrl: 'https://acme-provider.example.com/directory-url',
    accountKey: accountPrivateKey,
    externalAccountBinding: {
        kid: 'YOUR-EAB-KID',
        hmacKey: 'YOUR-EAB-HMAC-KEY'
    }
});
```

### Specifying the account URL

During the ACME account creation process, the server will check the supplied account key and either create a new account if 
the key is unused, or return the existing ACME account bound to that key.

In some cases, for example with some EAB providers, this account creation step may be prohibited and might require you to 
manually specify the account URL beforehand. This can be done through `accountUrl` in the client constructor.

```js
const client = new letsencrypt.Client({
    directoryUrl: acme.directory.letsencrypt.staging,
    accountKey: accountPrivateKey,
    accountUrl: 'https://acme-v02.api.letsencrypt.org/acme/acct/12345678'
});
```

You can fetch the clients current account URL, either after creating an account or supplying it through the constructor, using `getAccountUrl()`:

```js
const myAccountUrl = letsencrypt.getAccountUrl();
```


## Cryptography

For key pairs `letsencrypt/acme-client` utilizes native cryptography APIs, supporting signing and generation of both RSA and ECDSA keys. 
The [TextEncoder](https://google.com/search?query="TextEncoder") is used to generate and parse Certificate Signing Requests.

utility methods are exposed through `letsencrypt.crypto`.

* __Documentation: [docs/crypto.md](docs/crypto.md)__

```js

const privateRsaKey = await letsencrypt.crypto.createPrivateRsaKey();
const privateEcdsaKey = await letsencrypt.crypto.createPrivateEcdsaKey();

const [certificateKey, certificateCsr] = await letsencrypt.crypto.createCsr({
    commonName: '*.example.com',
    altNames: ['example.com']
});
```


### Legacy `.crypto` interface
Usage of crypto.subtle directly is highly prefered
You should consider migrating to the new `.crypto` API at your earliest convenience. More details can be found in the [letsencrypt/acme-client v6 upgrade guide](docs/wip-v6.md).

### Legacy `.forge` interface

The legacy `node-forge` crypto interface is still available for backward compatibility, however this interface is now considered deprecated and will be removed in a future major version of `acme-client`.

You should consider migrating to the new `.crypto` API at your earliest convenience. More details can be found in the [acme-client v5 upgrade guide](docs/upgrade-v5.md).

* __Documentation: [docs/forge.md](docs/forge.md)__


## Auto mode

For convenience an `auto()` method is included in the client that takes a single config object. This method will handle the entire process of getting a certificate for one or multiple domains.

* __Documentation: [docs/client.md#AcmeClient+auto](docs/client.md#AcmeClient+auto)__
* __Full example: [examples/auto.js](examples/auto.js)__

```js
const autoOpts = {
    csr: '<PEM encoded CSR>',
    email: 'test@example.com',
    termsOfServiceAgreed: true,
    challengeCreateFn: async (authz, challenge, keyAuthorization) => {},
    challengeRemoveFn: async (authz, challenge, keyAuthorization) => {}
};

const certificate = await client.auto(autoOpts);
```


### Challenge priority

When ordering a certificate using auto mode, `letsencrypt/acme-client` uses a priority list when selecting challenges to respond to. Its default value is `['http-01', 'dns-01']` which translates to "use `http-01` if any challenges exist, otherwise fall back to `dns-01`".

While most challenges can be validated using the method of your choosing, please note that __wildcard certificates can only be validated through `dns-01`__. More information regarding Let's Encrypt challenge types [can be found here](https://letsencrypt.org/docs/challenge-types/).

To modify challenge priority, provide a list of challenge types in `challengePriority`:

```js
await client.auto({
    ...,
    challengePriority: ['http-01', 'dns-01']
});
```


### Internal challenge verification

When using auto mode, `acme-client` will first validate that challenges are satisfied internally before completing the challenge at the ACME provider. In some cases (firewalls, etc) this internal challenge verification might not be possible to complete.

If internal challenge validation needs to travel through an HTTP proxy, see [HTTP client defaults](#http-client-defaults).

To completely disable `acme-client`s internal challenge verification, enable `skipChallengeVerification`:

```js
await client.auto({
    ...,
    skipChallengeVerification: true
});
```


## API

For more fine-grained control you can interact with the ACME API using the methods documented below.

* __Documentation: [docs/client.md](docs/client.md)__
* __Full example: [examples/api.js](examples/api.js)__

```js
const account = await client.createAccount({
    termsOfServiceAgreed: true,
    contact: ['mailto:test@example.com']
});

const order = await client.createOrder({
    identifiers: [
        { type: 'dns', value: 'example.com' },
        { type: 'dns', value: '*.example.com' }
    ]
});
```

## Debugging

To get a better grasp of what `letsencrypt/acme-client` is doing behind the scenes, you can either pass it a logger function, 
or enable debugging through an environment variable.

Setting a logger function may for example be useful for passing messages on to another logging system, or just dumping them to the console.

```js
letsencrypt.setLogger((message) => {
    console.log(message);
});
```

## The Unlicense - License

[Unlicense](LICENSE)
