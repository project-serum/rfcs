- Start Date: 2020-10-19
- RFC PR: 6
- Issue: N/A

# Summary
[summary]: #summary

Define an encrypted messaging protocol on Solana.

# Motivation
[motivation]: #motivation

Create a standard to enable multiple messaging clients to send encrypted messages.

# Design
[design]: #design

## Overview

There are two types of messages defined in this standard. 
1. Encrypted messages.
2. Presence messages.

A DAPP implementing this standard should be able to write both messages to the data field of an account on chain.

## Message
**Message Structure**
[512 Encrypted Bytes,512 Encrypted Bytes,4 Reserved Bytes]
Total Length: 1028 Bytes

A message is a JSON object string with a text message from a user, a 5 character universally unique identifier (uuid), and the signature of the uuid using Solana public key.

**Message**
```
Message = JSON.stringify({
	t:txt_msg, 
	u:uuid,
	us:uuid_signature
})
```

The message string should then be encrypted with the public key of an RSA key pair with a modulus length of 4096.  Due to the overhead when encrypting with an RSA public key of modules length 4096 the maximum length of a string encrypted is under 512 characters. Therefore the message
should be split in to two parts and then encrypted.

The method to split the message may vary by client but each piece should include enough entropy as not to allow an attacker to brute force an encrypted message to reveal the message recipient.

**Encrypted_Message**
```
Encrypted_Message = Encrypt(Message[0,x]) + Encrypt(Message[x,message.Length]) + [null,null,null,null]
```
A decrypted message is the concatenation of the the decrypted data field of the account

```
Unencrypted_Message = Decrypt(data[0,512]) + Decrypt(data[512,1024])
```

## Presence Message
**Presence Structure**
[1028 Bytes]

A signaling message to bind a Solana public key with a RSA public key.
A space separated message consisting of the user's Solana public key, the user's RSA public key, and the signature of the previous two items combined with a space separator.
The signature in the presence message should be created with the user's Solana private key.

```
Presence_Message = Solana_Address + " "+ RSA_PublicKey+ " " + Signature(Solana_Address + " "+ RSA_PublicKey)
```
The message should be padded will null bytes.






# Interface
[design]: #Interface

Reference implementation in c

```
#include <solana_sdk.h>
#include <deserialize_deprecated.h>

extern uint64_t entrypoint(const uint8_t *input) {
  SolAccountInfo accounts[1];
  SolParameters params = (SolParameters){.ka = accounts};

  if (!sol_deserialize_deprecated(input, &params, SOL_ARRAY_SIZE(accounts))) {
    return ERROR_INVALID_ARGUMENT;
  }
  
  //Enforce message size
  if(params.data_len != 1028){
	  sol_log("Input data incorrect size");
	  return ERROR_INVALID_ARGUMENT;
  }
  
  for(int i = 1027; i > -1;i--){params.ka[0].data[i] = params.data[i];}
  return SUCCESS;
}
```


## Flow

#### Announcement
1. The UX allows a user to create a RSA key pair with modulus length 4096.
3. The UX interacts with the Program to write a presence message to the account data field.
4. The user is prompted to sign and send the transaction.

#### Message
1. The UX acquires the text value of a user's message.
2. The client generates a  uuid and prompts the user to sign the uuid with their Solana public key.
3. The client use's the public key from a previous announcement to generate an encrypted message.
4. The UX interacts with the Program to write an encrypted message to the account data field.
5. The user is prompted to sign and send the transaction.

A client listening to changes in the Account data attempts to decrypt the encrypted message or parse a presence message.

# Rationale and Alternatives

**Why use Sollet wallet to sign transactions?** 
It is safer for the user to use a trusted wallet account rather than import their keys, however there is more friction if the wallet does not understand the transaction the user is trying to sign.
Sollet wallet does not "auto allow" transactions it cannot parse.
Work with the Sollet wallet team needs to be done to allow unique transaction interactions.
It is also better to give the users the option to import a private key or create a new account to bypass signature prompts.

**Why fixed length messages?**
 Longer messages increase the complexity of brute forcing to reveal the recipient of a message.
 
**Why include uuid and uuid_signature?**
To allow for the fact the user interacting with the smart contract on chain may not be the original author of the message.

**Why 1028 Bytes messages?** 
To pack the most amount of information available in a transaction in a message.

**Why reserve 4 Bytes?**
 They can be used for potentially versioning or signaling metadata about the message.

# Unresolved questions
[unresolved]: #unresolved-questions

1. Does it make sense to try to eliminate presence messages by only using Solana public key to encrypt messages?

