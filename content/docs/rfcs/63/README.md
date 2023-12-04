---
slug: 63
title: 63/STATUS-Keycard-Usage
name: Status Keycard Usage
status: raw
category: Standards Track
description: Describes how an application can use the Status Keycard to create, store, and transact with different accounts.
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
  - Jimmy Debe <jimmy@status.im>
---

# Terminology

- **Account**: A valid [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) compliant key.
- **Multiaccount**: An account from which multiple Accounts can be derived.

# Abstract

This specification describes how an application can use the Status Keycard to -

1. Create Multiaccounts
2. Store Multiaccounts
3. Use Multiaccounts for transaction or message signing
4. Derive Accounts from Multiaccounts

More documentation on the Status Keycard can be found [here](https://keycard.tech/docs/)

# Motivation

The Status Keycard is a hardware wallet that can be used to store and sign transactions.
For the purpose of the Status App, this specification describes how the Keycard SHOULD be used to store and sign transactions.

# Usage

## Endpoints

### 1. Initialize Keycard (`/init-keycard`)

To initialize the keycard for use with the application.
The keycard is locked with a 6 digit pin.

#### Request wire format

```json
{
  "pin": 6_digit_pin
}
```

#### Response wire format

```json
{
  "password": password_to_unlock_keycard,
  "puk": 12_digit_recovery_code,
  "pin": provided_pin,
}
```

The keycard MUST be initialized before it can be used with the application.
The application SHOULD provide a way to recover the keycard in case the pin is forgotten.

### 2. Get Application Info (`/get-application-info`)

To fetch if the keycard is ready to be used by the application.

#### Request wire format

The requester MAY add a `pairing` field to filter through the generated keys
```json
{
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt> OR null
}
```

#### Response wire format

##### If the keycard is not initialized yet

```json
{
  "initialized?": false
}
```

##### If the keycard is initialized

```json
{
  "free-pairing-slots": number, 
  "app-version": major_version.minor_version, 
  "secure-channel-pub-key": valid_bip32_key,, 
  "key-uid": unique_id_of_the_default_key,
  "instance-uid": unique_instance_id, 
  "paired?": bool,
  "has-master-key?": bool, 
  "initialized?" true
}
```

### 3. Pairing the Keycard to the Client device (`/pair`)

To establish a secure communication channel described [here](https://keycard.tech/docs/apdu/opensecurechannel.html), the keycard and the client device need to be paired.

#### Request wire format

```json
{
  "password": password_to_unlock_keycard
}
```

#### Response wire format

```json
"<shared_secret>/<pairing_index>/<256_bit_salt>"
```

### 4. Generate a new set of keys (`/generate-and-load-keys`)

To generate a new set of keys and load them onto the keycard.

#### Request wire format

```json
{
  "mnemonic": 12_word_mnemonic,
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>,
  "pin": 6_digit_pin
}
```

#### Response wire format

```json
{
  "whisper-address": 20_byte_whisper_compatible_address, 
  "whisper-private-key": whisper_private_key, 
  "wallet-root-public-key": 256_bit_wallet_root_public_key, 
  "encryption-public-key": 256_bit_encryption_public_key,, 
  "wallet-root-address": 20_byte_wallet_root_address, 
  "whisper-public-key": 256_bit_whisper_public_key,
  "address": 20_byte_address, 
  "wallet-address": 20_byte_wallet_address,, 
  "key-uid": 64_byte_unique_key_id, 
  "wallet-public-key": 256_bit_wallet_public_key,
  "public-key": 256_bit_public_key,
  "instance-uid": 32_byte_unique_instance_id,
}
```

### 5. Get a set of generated keys (`/get-keys`)

To fetch the keys that are currently loaded on the keycard.

#### Request wire format

```json
{
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>,
  "pin": 6_digit_pin
}
```

#### Response wire format

```json
{
  "whisper-address": 20_byte_whisper_compatible_address, 
  "whisper-private-key": whisper_private_key, 
  "wallet-root-public-key": 256_bit_wallet_root_public_key, 
  "encryption-public-key": 256_bit_encryption_public_key,
  "wallet-root-address": 20_byte_wallet_root_address, 
  "whisper-public-key": 256_bit_whisper_public_key,
  "address": 20_byte_address, 
  "wallet-address": 20_byte_wallet_address, 
  "key-uid": 64_byte_unique_key_id, 
  "wallet-public-key": 256_bit_wallet_public_key,
  "public-key": 256_bit_public_key,
  "instance-uid": 32_byte_unique_instance_id,
}
```

### 6. Sign a transaction (`/sign`)

To sign a transaction using the keycard, passing in the pairing information and the transaction to be signed.

#### Request wire format

```json
{
  "hash": 64_byte_hash_of_the_transaction,
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>,
  "pin": 6_digit_pin,
  "path": bip32_path_to_the_key
}
```

#### Response wire format

```json
<256_bit_signature>
```

### 7. Export a key (`/export-key`)

To export a key from the keycard, passing in the pairing information and the path to the key to be exported.

#### Request wire format

```json
{
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>,
  "pin": 6_digit_pin,
  "path": bip32_path_to_the_key
}
```

#### Response wire format

```json
<256_bit_public_key>
``` 

### 8. Verify a pin (`/verify-pin`)

To verify the pin of the keycard.

#### Request wire format

```json
{
  "pin": 6_digit_pin
}
```

#### Response wire format

```json
1_digit_status_code
```

Status code reference:
- 3: PIN is valid
<!--TODO: what are the other status codes?-->


### 9. Change the pin (`/change-pin`)

To change the pin of the keycard.

#### Request wire format

```json
{
  "new-pin": 6_digit_new_pin,
  "current-pin": 6_digit_new_pin,
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>
}
```

#### Response wire format

##### If the operation was successful

```json
true
```

##### If the operation was unsuccessful

```json
false
```

### 10. Unblock the keycard (`/unblock-pin`)

If the Keycard is blocked due to too many incorrect pin attempts, it can be unblocked using the PUK.

#### Request wire format

```json
{
  "puk": 12_digit_recovery_code,
  "new-pin": 6_digit_new_pin,
  "pairing": <shared_secret>/<pairing_index>/<256_bit_salt>
}
```

#### Response wire format

##### If the operation was successful

```json
true
```

##### If the operation was unsuccessful

```json
false
```

## Flows

Any application that uses the Status Keycard MAY implement the following flows according to the actions listed above.

### 1. A new user wants to use the Keycard with the application

1. The user initializes the Keycard using the `/init-keycard` endpoint.
2. The user pairs the Keycard with the client device using the `/pair` endpoint.
3. The user generates a new set of keys using the `/generate-and-load-keys` endpoint.
4. The user can now use the Keycard to sign transactions using the `/sign` endpoint.

### 2. An existing user wants to use the Keycard with the application

1. The user pairs the Keycard with the client device using the `/pair` endpoint.
2. The user can now use the Keycard to sign transactions using the `/sign` endpoint.

### 3. An existing user wants to use the Keycard with a new client device

1. The user pairs the Keycard with the new client device using the `/pair` endpoint.
2. The user can now use the Keycard to sign transactions using the `/sign` endpoint.

### 4. An existing user wishes to verify the pin of the Keycard

1. The user verifies the pin of the Keycard using the `/verify-pin` endpoint.

### 5. An existing user wishes to change the pin of the Keycard

1. The user changes the pin of the Keycard using the `/change-pin` endpoint.

### 6. An existing user wishes to unblock the Keycard

1. The user unblocks the Keycard using the `/unblock-pin` endpoint.


# Security Considerations

Inherits the security considerations of [Status Keycard](https://keycard.tech/docs/)

# Privacy Considerations

Inherits the privacy considerations of [Status Keycard](https://keycard.tech/docs/)


# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [BIP-32 specification](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
2. [Keycard documentation](https://keycard.tech/docs/)
3. [16/Keycard-Usage](https://specs.status.im/draft/16)
