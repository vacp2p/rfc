---
slug: 63
title: 63/STATUS-Keycard-Usage
name: Status Keycard Usage
status: raw
category: Standards Track
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
  - "?"
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

```json
{}
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

## Flows

## Key storage

# Security Considerations

# Privacy Considerations


# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [BIP-32 specification](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
2. [Keycard documentation](https://keycard.tech/docs/)