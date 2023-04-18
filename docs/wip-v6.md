# Upgrading to v6

This document outlines the breaking changes introduced in v5 of `acme-client`, why they were introduced and what you should look out for when upgrading 
your application. First off this release adds support for Chromium, and the reason for that is a new native crypto interface - more on that below. 

## New native crypto interface

A new crypto interface has been introduced with v6, which you can find under `crypto`. It uses native cryptography APIs to generate private keys, JSON Web Keys and signatures, and finally enables support for ECC/ECDSA (P-256, P384 and P521), both for account private keys and certificates. The [jsrsasign](https://www.npmjs.com/package/jsrsasign) module is used to handle generation and parsing of Certificate Signing Requests.

Full documentation of `crypto` can be [found here](crypto.md).

All API's do always Return Promises as we never know where the result comes from.

Below you will find a table summarizing the current `acme.forge` `acme-client` methods, and their new `crypto` replacements. 
A summary of the changes for each method, including examples on how to migrate, can be found following the table.

*Note: The now deprecated `acme.forge` àcme-client` interface is still available for use in v5, and is removed, 
Should you not wish to change to the new interface right away, the following breaking changes will immediately affect you.*

- :green_circle: = API functionality unchanged between `acme.forge` and `acme.crypto`
- :orange_circle: = Slight API changes, like depromising or renaming, action may be required
- :red_circle: = Breaking API changes or removal, action required if using these methods

| Deprecated `.forge` API       | New `.crypto` API             | State                 | v6 |
| ----------------------------- | ----------------------------- | --------------------- | ----
| `await createPrivateKey()`    | `await createPrivateKey()`    | :green_circle:        | await privateKey
| `await createPublicKey()`     | `await getPublicKey()`        | :green_circle:   (1)  | await publicKey
| `await getModulus()`          | `await getJwk()`              | :red_circle:      (3) | await jwk
| `await getPublicExponent()`   | `await getJwk()`              | :red_circle:      (3) | 
| `await readCsrDomains()`      | `await readCsrDomains()`      | :green_circle:   (4)  | await certificateDomains
| `await readCertificateInfo()` | `await readCertificateInfo()` | :green_circle:   (4)  | await certificateInfo
| `getPemBody()`                | `await getPemBodyAsB64u()`    | :red_circle:      (2) | await certificateBody
| `splitPemChain()`             | `await splitPemChain()`       | :green_circle:        | await splitCertificateBody
| `await createCsr()`           | `await createCsr()`           | :green_circle:        | await createCertificate


### 1. `createPublicKey` renamed and depromised

* The method `createPublicKey()` has been renamed to `getPublicKey()`

```js
// Before
const publicKey = await acme.forge.createPublicKey(privateKey);

// After
const publicKey = await acme.crypto.getPublicKey(privateKey);
```


### 2. `getPemBody` renamed, now returns Base64URL

* Method `getPemBody()` has been renamed to `getPemBodyAsB64u()`
* Instead of a Base64-encoded PEM body, now returns a Base64URL-encoded PEM body
* :red_circle: **This is a breaking change**

```js
// Before
const body = await acme.forge.getPemBody(pem);
const body = await acme.crypto.getPemBodyAsB64u(pem);

// After
// @type {URL} A Base64u String 
const body = await acme.crypto.certificateBody(pem);
```

### 3. `getModulus` and `getPublicExponent` merged into `getJwk`

* Methods `getModulus()` and `getPublicExponent()` have been removed
* Replaced by new method `getJwk()`
* :red_circle: **This is a breaking change**

```js
// Before
const mod = await acme.forge.getModulus(key);
const exp = await acme.forge.getPublicExponent(key);

// After
const { e, n } = await acme.crypto.jwk(key);
```

### 4. `readCsrDomains` and `readCertificateInfo` are now certificateInfo

* Methods `readCsrDomains()` and `readCertificateInfo()` 

```js
// Before
const domains = await acme.forge.readCsrDomains(csr);
const info = await acme.forge.readCertificateInfo(certificate);

// After
const domains = await acme.crypto.certificateDomains(csr);
const info = await acme.crypto.certificateInfo(certificate);
```
