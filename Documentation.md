# Pass Butler documentation

Author: Bastian Raschke <bastian.raschke@posteo.de>

## Features

- Synchronisation to server
- End-to-end encryption (server compromise does not compromises user data)
- Share passwords with users on same server instance
- Biometric unlock
- Autolock after amount of time - if locked, no sensitive data is stored in memory
- Flat list of entries (no groups) - search with name/tags
- Modern UI (Material design, supports dark mode)
- Android Autofill system service

## Password entry

- Icon
- Username
- Password
- URL
- Notes
- Tags (for search)

## Synchronisation

All model entities are identified via UUID (no auto increment ID to be sure clients that are not always synced do not create conflicts). The up-to-dateness of an model entity is determined via modified timestamp. Model entities are never deleted in database - instead they contain a deleted field - this makes new/deleted detection much more simple.

The synchronisation process:

1) Load list of local entities
2) Load list of remote entities
3) Detect new local entities and insert locally
4) Detect new remote entities and insert remotely
5) Detect modified local entities (according to modified field) and update locally
6) Detect modified remote entities (according to modified field) and update remotely

## Model entities

### User

- **username**
- masterPasswordAuthenticationHash
- masterKeyDerivationInformation
- masterEncryptionKey (protected)
- itemEncryptionPublicKey
- itemEncryptionSecretKey (protected)
- settings (protected)
- deleted
- modified
- created

### Item

- **id**
- username (owner / creator)
- itemdata (actual item data)
- deleted
- modified
- created

### Item Authorization

- **id**
- username
- item_id
- itemkey (protected with users `Item Encryption Key Pair` public key)
- readonly
- deleted
- modified
- created

## Security architecture

### Cryptografic entities

#### `Master Password`

- Benutzerpasswort

#### `Master Key`

- Ableitung: PBKDF2-SHA256(`Master Password`, Salt, Iterations=100000)
- muss nicht gespeichert werden (wird durch `Master Password` abgeleitet)

#### `Master Encryption Key`

- symmetrischer AES 256 Key
- zur Verschlüsselung von geschützen Nutzerdaten
- geschützt/verschlüsselt mit `Master Key`
- bleibt immer gleich (wird bei Änderung des `Master Key` neu verschlüsselt)
- geschützen Nutzerdaten werden nicht direkt mit `Master Key` verschlüsselt, um bei einer Passwortänderung die potentiell fehlerintensive Neuverschlüsselung aller einzelnen Nutzerdaten zu vermeiden)

#### `Item Encryption Key Pair`

- asymmetrisches RSA 2048-Schlüsselpaar
- zur Verschlüsselung von Item Keys welche die `Item.data` schützen
- nötig damit Items anderen Nutzern geteilt werden können (neuer `ItemAuthorization` für neuen Nutzer wird angelegt und der `itemkey` mit dessen öffentlichen Schlüssel verschlüsselt)

## Server API

### Authentication

The plain master password must NEVER sent to server to ensure E2E encryption: if the server get compromised, the attacker can't decrypt the user data if it is only possible for him to eavedrop the authentication passwords but NOT the master passwords of the users. So the master password is first hashed (PBKDF2-SHA256 with 100001 iterations - one iteration more than for `Master Key` derivation - using the username as the salt) on client side and send to the server where it is hashed again (PBKDF2-SHA256 with 150000 iterations - 50k iterations more to more slow down computing time). It is hashed on server again to avoid the "hash-is-the-password" situation where an attacker can straight-forward authenticate with stolen hashes.

### Server resources

#### GET /v1/token

- zeitlich beschränktes Access-Token, welches mit Benutzer/Passwort abholbar ist für einen nicht-gelöscht-flagged Benutzer

#### GET /v1/users

- Liste von allen Usern (beschränkte Felder)

#### PUT /v1/users

- Allows to "register" a local user to become a remote user on server - only possible if:
  - Configuration value `ENABLE_REGISTRATION` is set to `True`
  - User does not exists in database

#### GET /v1/user

- Lesen aller Felder des eigenen Benutzers

#### PUT /v1/user

- Schreiben veränderlicher Felder des eigenen Benutzers

#### GET /v1/user/itemauthorizations

- Liste aller:
  - Itemauthorizations, auf die der eigene Benutzer Zugriff hat (this.userId == currentUserId) 
  - Itemauthorizations, der der aktuelle Benutzer an andere Benutzer ausgestellt hat (this.itemId.userId == currentUserId)

Hinweis: Itemauthorizations, die gelöscht-flagged wurden, werden auch gelistet, damit entsprechende Benutzer diese syncen können.

#### PUT /v1/user/itemauthorizations

- akzeptiert Liste von Itemauthorizations, dessen Item dem eigenen Benutzer gehört (this.itemId.userId == currentUserId) - nur der Autor eines Item kann Itemauthorizations erstellen/verändern
- ausschließlich Schreiben veränderlicher Felder

Ablauf Berechtigung wird entzogen:
- entsprechende Itemauthorization wird gelöscht-flagged
- entsprechender Benutzer kann aktuelle Version vom Item nicht mehr laden (lokale Kopie aber weiterhin verfügbar auf dessen Datenstand)

#### GET /v1/user/items

- Liste von Items für die der eigene Benutzer eine nicht-gelöscht-flagged Itemauthorization besitzt

Hinweis: Items, die gelöscht-flagged wurden, werden auch gelistet, damit entsprechende Benutzer diese Information syncen können, solange sie eine nicht-gelöscht-flagged Itemauthorization haben. Der gelöscht-flagged Status soll wie jede andere Veränderung des Items wirksam und sichtbar sein für alle zugelassenen Benutzer. 

#### PUT /v1/user/items

- akzeptiert Liste von Items für die der eigene Benutzer eine nicht-gelöscht-flagged und "read only" = False Itemauthorization ausgestellt bekommen hat
- ausschließlich Schreiben veränderlicher Felder
