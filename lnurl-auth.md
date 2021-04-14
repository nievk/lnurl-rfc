# LNURL-auth


## Authorization with Bitcoin Wallet

A special `linkingKey` can be used to login user to a service or authorise sensitive actions. This preferrably should be done without compromising user identity so plain LN node key can not be used here. Instead of asking for user credentials a service could display a "login" QR code which contains a specialized `LNURL`.


### Server-side generation of auth URL and signature verification:

When creating an `LNURL-auth` handler `LN SERVICE` must include in it a `k1` query parameter consisting of randomly generated 32 bytes of data as well as optional `action` enum, an example is `https://site.com?tag=login&k1=hex(32 bytes of random data)&action=login`.

Later, once `LN SERVICE` receives a call at the specified `LNURL-auth` handler, it should take `k1`, `key` and a DER-encoded `sig` and verify the signature using `secp256k1`. Once signature is successfully verified a user provided `key` can be used as identifier and may be stored in a session, database or however `LN SERVICE` sees fit.

`LN SERVICE` must make sure that unexpected `k1`s are not accepted: it is strongly advised for `LN SERVICE` to have a cache of unused `k1`s, only proceed with verification of `k1`s present in that cache and remove used `k1`s on successful auth attempts.

### Server-side choice of subdomain:

 `LN SERVICE` should carefully choose which subdomain (if any) will be used as LNURL-auth endpoint and stick to chosen subdomain in future. For example, if `auth.site.com` was initially chosen then changing it to, say, `login.site.com` will result in different account for each user because full domain name is used by wallets as material for key derivation.

 `LN SERVICE` should consider giving meaningful names to chosen subdomains since `LN WALLET` may show a full domain name to users on login attempt. For example, `auth.site.com` is less confusing than `ksf03.site.com`.


### Wallet to service interaction flow:

1. `LN WALLET` scans a QR code and decodes an URL which is expected to have the following query parameters:
    - `tag` with value set to `login` which means no GET should be made yet.
    - `k1` (hex encoded 32 bytes of challenge) which is going to be signed by user's `linkingPrivKey`.
    - optional `action` enum which can be one of four strings: `register | login | link | auth`.
2. `LN WALLET` displays a "Login" dialog which must include a domain name extracted from `LNURL` query string and `action` enum translated into human readable text if `action` query parameter was present.
3. Once accepted by user, `LN WALLET` signs `k1` on `secp256k1` using `linkingPrivKey` and DER-encodes the signature. `LN WALLET` Then issues a GET to `LN SERVICE` using `<LNURL_hostname_and_path>?<LNURL_existing_query_parameters>&sig=<hex(sign(utf8ToBytes(k1), linkingPrivKey))>&key=<hex(linkingPubKey)>`
4. `LN SERVICE` responds with the following JSON once client signature is verified:
    ```
    {
        status: "OK"
    }
    ```
    or

    ```
    {"status": "ERROR", "reason": "error details..."}
    ```

`action` enums meaning:
- `register`: service will create a new account linked to user's `linkingKey`.
- `login`: service will login user to an existing account linked to user's `linkingKey`.
- `link` service will link a user provided `linkingKey` to user's existing account (if account was not originally created using `lnurl-auth`).
- `auth`: some stateless action which does not require logging in (or possibly even prior registration) will be granted.

## `linkingKey` derivation

`LNURL-auth` works by deriving domain-specific `linkingKey`s from user seed. This approach has two goals: first one is simplicity (user only needs to keep mnemonic to preserve both funds and identity), second one is portability (user should be able to switch a wallet by entering the same mnemonic and get the same identity).

However, second goal is not reachable in practice because there exist different formats of seeds which can't be transferred across all existing wallets. As such a practical approach is to have a recommended ways to derive `linkingKey` for different wallet types.

### `linkingKey` derivation for BIP-32 based wallets:

1. There exists a private `hashingKey` which is derived by user `LN WALLET` using `m/138'/0` path.
2. `LN SERVICE` full domain name is extracted from login `LNURL` and then hashed using `hmacSha256(hashingKey, full service domain name)`. Full domain name here means FQDN with last comma omitted (Example: for `https://x.y.z.com/...` it would be `x.y.z.com`).
3. First 16 bytes are taken from resulting hash and then turned into a sequence of 4 `Long` values which are in turn used to derive a service-specific `linkingKey` using `m/138'/<long1>/<long2>/<long3>/<long4>` path, a Scala example:

```Scala
import fr.acinq.bitcoin.crypto
import fr.acinq.bitcoin.Protocol
import java.io.ByteArrayInputStream
import fr.acinq.bitcoin.DeterministicWallet._
val domainName = "site.com"
val hashingPrivKey = derivePrivateKey(walletMasterKey, hardened(138L) :: 0L :: Nil)
val derivationMaterial = hmac256(key = hashingPrivKey.toBin, message = domainName)
val stream = new ByteArrayInputStream(derivationMaterial.slice(0, 16).toArray)
val pathSuffix = Vector.fill(4)(Protocol.uint32(stream, ByteOrder.BIG_ENDIAN)) // each uint32 call consumes next 4 bytes
val linkingPrivKey = derivePrivateKey(walletMasterKey, hardened(138L) +: pathSuffix)
val linkingPubKey = linkingPrivKey.publicKey
```

### `linkingKey` derivation for wallets which don't have an access to master `privKey`:

In this case neither `hashingKey` nor domain-specific `linkingKey`s can be derived by path. To overcome this limitation a different scheme is used for this class of wallets:

1. The following canonical phrase is defined: `DO NOT EVER SIGN THIS TEXT WITH YOUR PRIVATE KEYS! IT IS ONLY USED FOR DERIVATION OF LNURL-AUTH HASHING-KEY, DISCLOSING ITS SIGNATURE WILL COMPROMISE YOUR LNURL-AUTH IDENTITY AND MAY LEAD TO LOSS OF FUNDS!`.
2. `LN WALLET` obtains an `RFC6979` deterministic signature of `sha256(utf8ToBytes(canonical phrase))` using `secp256k1` with node private key.
3. `LN WALLET` defines `hashingKey` as `PrivateKey(sha256(obtained signature))`.
4. `SERVICE` domain name is extracted from auth `LNURL` and then service-specific `linkingPrivKey` is defined as `PrivateKey(hmacSha256(hashingKey, service domain name))`.

`LN WALLET` must make sure it is not possible to accidentally or automatically sign and hand out a signature of canonical phrase.
