# Pass Butler documentation

Pass Butler is a password manager which provides a private cloud solution to synchronize the user data to be able to use it on multiple devices easily. The user data is end-to-end (E2E) encrypted to ensure the data can't even read by server administrator. Additionally Pass Butler offers password sharing which means a user can grant access to selected passwords to other user on the server (e.g. to share Wifi passwords in a team). This involves techniques which will be documented in this file.

## Model entities

### User

| Column                           | Encrypted | Description                                                                                                            |
|:---------------------------------|:----------|:-----------------------------------------------------------------------------------------------------------------------|
| id                               | no        | The primary key of the entity (UUID)                                                                                   |
| username                         | no        | A unique user identifying string                                                                                       |
| masterPasswordAuthenticationHash | no        | The [Server Authentication Hash](#server-authentication-hash)                                                          |
| masterKeyDerivationInformation   | no        | A random salt and iteration count to derive the [Master Key](#master-key) from the [Master Password](#master-password) |
| masterEncryptionKey              | yes       | The [Master Encryption Key](#master-encryption-key)                                                                    |
| itemEncryptionPublicKey          | no        | The public part of the [Item Encryption Key Pair](#item-encryption-key-pair)                                           |
| itemEncryptionSecretKey          | yes       | The private part of the [Item Encryption Key Pair](#item-encryption-key-pair)                                          |
| settings                         | yes       | The user settings                                                                                                      |
| deleted                          | no        | Indicates if the entity was deleted                                                                                    |
| modified                         | no        | The unix timestamp of last modification                                                                                |
| created                          | no        | The unix timestamp of creation                                                                                         |

### Item

| Column   | Encrypted | Description                                                        |
|:---------|:----------|:-------------------------------------------------------------------|
| id       | no        | The primary key of the entity (UUID)                               |
| username | no        | The creator / owner of the item                                    |
| itemData | yes       | Contains actual data of the item (username, password, notes, etc.) |
| deleted  | no        | Indicates if the entity was deleted                                |
| modified | no        | The unix timestamp of last modification                            |
| created  | no        | The unix timestamp of creation                                     |

### Item Authorization

| Column   | Encrypted | Description                                                   |
|:---------|:----------|:--------------------------------------------------------------|
| id       | no        | The primary key of the entity (UUID)                          |
| userId   | no        | The user id that can use the authorization to access the item |
| itemId   | no        | The item id to which was granted access to                    |
| itemKey  | yes       | The symmetric key to decrypt the item data                    |
| readOnly | no        | Indicates if the authorization to access the item is readonly |
| deleted  | no        | Indicates if the entity was deleted                           |
| modified | no        | The unix timestamp of last modification                       |
| created  | no        | The unix timestamp of creation                                |

## Cryptographic algorithms

### PBKDF2-SHA256 {#pbkdf2-sha256}

A key derivation algorithm that uses SHA-256. It needs a salt and an iteration count to slow down computing time (brute forcing).

### AES-256-GCM {#aes-256-gcm}

A symmetric encryption algorithm with a key length of 256 bit in Galois/Counter mode (GCM) that ensures not only the privacy of the data but also the authentication to protect against tampering the encrypted data. A random initialization vector (IV) that must never recycled is needed for the block mode. The algorithm is very fast and suitable for all kinds of data amount.

### RSA-2048-OAEP {#rsa-2048-oaep}

An asymmetric encryption algorithm with a key length of 2048 bit that consists of a public and a private/secret part. The public part allows to encrypt data, the private part allows to decrypt the data. The algorithm is slow and only suitable for small data.

## Cryptographic entities

### Master Password {#master-password}

The user password that protects all other data. It should be long and complex because the complete security relies on it! It is stored in memory only temporary for computing and is overridden afterwards immediately.

### Master Key {#master-key}

The symmetric encryption/description key that is derived from the [Master Password](#master-password) with [PBKDF2-SHA256](#pbkdf2-sha256) with an iteration count and a random salt stored in `User.masterKeyDerivationInformation`. Like the [Master Password](#master-password), it is derived and stored in memory only temporary for computing and is overridden afterwards immediately.







### Master Encryption Key

`User.masterEncryptionKey`


- symmetrischer AES 256 Key
- zur Verschlüsselung von geschützen Nutzerdaten
- geschützt/verschlüsselt mit `Master Key`
- bleibt immer gleich (wird bei Änderung des `Master Key` neu verschlüsselt)
- geschützen Nutzerdaten werden nicht direkt mit `Master Key` verschlüsselt, um bei einer Passwortänderung die potentiell fehlerintensive Neuverschlüsselung aller einzelnen Nutzerdaten zu vermeiden)

### Item Encryption Key Pair {#item-encryption-key-pair}

- asymmetrisches RSA 2048-Schlüsselpaar
- zur Verschlüsselung von Item Keys welche die `Item.data` schützen
- nötig damit Items anderen Nutzern geteilt werden können (neuer `ItemAuthorization` für neuen Nutzer wird angelegt und der `itemkey` mit dessen öffentlichen Schlüssel verschlüsselt)



### Local Authentication Hash {#local-authentication-hash}

This hash is used to authenticate on the server together with the username. It is a replacement for a classic username and password authentication to ensure the [Master Password](#master-password) never ever leaves the local client to ensure E2E encryption but also proves, that the user knows the master password. It is derived with [PBKDF2-SHA256](#pbkdf2-sha256) using the username as the salt and 100001 iterations (one iteration more than for [Master Key](#master-key) derivation to differ from it).

### Server Authentication Hash {#server-authentication-hash}

This hash 


## Server authentication


The plain master password must NEVER sent to server to ensure E2E encryption: if the server get compromised, the attacker can't decrypt the user data if it is only possible for him to eavesdrop the authentication passwords but NOT the master passwords of the users. So the master password is first hashed with [PBKDF2-SHA256](#pbkdf2-sha256) using the username as the salt and 100001 iterations (one iteration more than for [Master Key](#master-key) derivation) on client side and send to the server where it is hashed again ([PBKDF2-SHA256](#pbkdf2-sha256) with 150000 iterations - around 50k iterations more to more slow down computing time). It is hashed on server again to avoid the "hash-is-the-password" situation where an attacker can straight-forward authenticate with stolen hashes.

Token request process:

1) Client requests token with username and [Local Authentication Hash](#local-authentication-hash)
2) Server hashes the received [Local Authentication Hash](#local-authentication-hash) again with [PBKDF2-SHA256](#pbkdf2-sha256) using the random salt and iteration count stored in `User.masterPasswordAuthenticationHash`. If the calculated hash matches the hash value also stored in `User.masterPasswordAuthenticationHash`, the client is authenticated and the server responds a valid bearer token (JWT)

All later requests are authenticated with the bearer token to avoid the procedure for every request.





## Synchronization process

All model entities are identified via UUID (no auto increment ID to be sure clients that are not always synced do not create conflicts). The up-to-dateness of an model entity is determined via modified timestamp. Model entities are never deleted in database - instead they contain a deleted field - this makes new/deleted detection much more simple.

The following steps are executed for all model entities:

1) Load list of local entities
2) Load list of remote entities
3) Detect new local entities and insert locally
4) Detect new remote entities and insert remotely
5) Detect modified local entities (according to modified field) and update locally
6) Detect modified remote entities (according to modified field) and update remotely
