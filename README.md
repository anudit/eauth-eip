---
eip:
title: Standards and Best Practices for implementing Sign-in with Ethereum.
author: [Anudit Nagar](https://github.com/anudit)
discussions-to:
status:
type: Standards Track
category: ERC
created:
requires: 137, 634, 1193, 2304
---

## Simple Summary
Defines the Standards and Best Practices for implementing Sign-in with Ethereum.

## Abstract

We introduce a standard for implementing Sign-in in with Ethereum to authenticate users on websites. This enables any website on the Internet to add support for Ethereum based sign-in using Json Web Tokens (JWT) for authentication with native support for existing Web3 Wallets like MetaMask, Walletconnect Et al. The standard expands on best practices to make Web3 native actions like Signing a message from an Ethereum Account understandable for users.

## Motivation

1. An Ethereum based Web3 Login to the Web2 world enabling any application to have support for Authentication via Decentralized Identities.
2. A Single Sign-On approach enabling users to have a single authentication method that works across all sites.
3. ENS enables users to register a Web3 username that wraps multiple identities across web2 and web3, can point to a decentralized website all without touching a single centralized service and then hold custody of it themselves in their Ethereum account.

## Specification

### Overview

To break it down the login flow,

1. The User clicks Sign-In with Ethereum and chooses a Wallet to connect.
2. The User connects the wallet by completing the wallet authentication process (automatically connects if already authenticated).
2. The User Signs a `Challenge Message` from the Wallet.
3. The Application verifies the User indeed owns the account and generates a JWT token authorizing the user.

### Definitions

OAuth (Open Authorization) is a standard that apps can use to provide client applications with "Authenticated Access". OAuth works over HTTPS and authorizes applications with access tokens rather than credentials

### Previous Literature

[EIP-1078](https://eips.ethereum.org/EIPS/eip-1078)
[EIP-2525](https://eips.ethereum.org/EIPS/eip-2525)

### Existing Implementations

 - [node-eauth-server](https://github.com/pelith/node-eauth-server)
 - [EthPress – Web3 Login](https://wordpress.org/plugins/ethpress/#developers)

### Implementation Specifications

The User Sign-In flow starts by showing users a "Sign-in with Ethereum" button. The UI for this should be similar to the standard Web2 Social login buttons,

![Sign-In with Ethereum](./images/signin-full.png "Sign-In with Ethereum")
![Sign-In with Ethereum](./images/signin-short.png "Sign-In with Ethereum")

Next, the user chooses the wallet they want to use. Wallet bundlers like Web3Modal, Onboard.js can be used to support multiple Web3 Wallet Providers out-of-the-box.

Next, the users sign the `Challenge Message`. This part can be intimidating to users who do not have an understanding of what signing a message means and is also a crucial security step that proves the user's ownership of the account. Hence, this message should be descriptive of its purpose and the access the application.user gets after signing.
For example, "Sign this message to prove the ownership of the address <eth-address> to <application-name>. Timestamp:<timestamp> Validity:<validity>".

Here a `timestamp` is used to make sure that a past signed message payload cannot be reused again, this can further be expanded to a nonce stored on the server that can be revoked by the user and forces re-authentication of all sessions.

`Validity` makes sure the user is aware of how long the session token is authorized for.


Server-sided validation can be implemented by 2 primary functions.

First to Recover an Address from a Signed Message Data (signature), Validate the address and issue a Session Token (JWT). E.g.,
```js
import jwt from 'jsonwebtoken';
import { recoverAddress, arrayify, hashMessage } from 'ethers/lib/utils';
let data = `Sign this message to prove the ownership of the address 0xc5F003779A46d494d32c558c1B280A9de6079273 to MockApp. Timestamp:1624018096329 Validity:1 Day`;
let recoveredAddress = recoverAddress(arrayify(hashMessage(data)), signature);
let currentTimestamp = Date.now();

if (currentTimestamp - req.body.timestamp < 60*1000 && recoveredAddress == req.body.signerAddress){
    let token = jwt.sign(
        {user: req.body.signerAddress},
        process.env.JWT_SECRET,
        { expiresIn: "1d" }
    );
}
```

Second to Validate the JWT token before making any server-side changes.
```js
import jwt from 'jsonwebtoken';
jwt.verify(req.body.token, process.env.JWT_SECRET, function(err, decoded) {
    if (err) {
        // return error
    }
    else {
        let {user, exp} = decoded;
        let now = Math.floor(Date.now()/1000);
        if (req.body.signerAddress === user && now < exp) {
            // Valid Session Token
        }
        else {
            // Invalid or Expired Session Token
        }
    }
});
```

Once the user is logged in, they and other users on the site can be identified by their ENS Address by doing a reverse lookup,
```js
import { ethers } from "ethers";
let provider = new ethers.providers.InfuraProvider("mainnet");
let ensAddress = await provider.lookupAddress(address);
```
or directly resolve mutiple ENS Addresses on chain using the below method below (It is faster to get the data in 1 call to the contract passing multiple addresses as a parameter, than sending multiple calls in parallel for resolution),
```js
let provider = new ethers.providers.InfuraProvider("mainnet");

const resolverAbi = [
    {
        inputs: [
        { internalType: 'address[]', name: 'addresses', type: 'address[]' }
        ],
        name: 'getNames',
        outputs: [{ internalType: 'string[]', name: 'r', type: 'string[]' }],
        stateMutability: 'view',
        type: 'function'
    }
];

const resolverAddress = "0x3671aE578E63FdF66ad4F3E12CC0c0d71Ac7510C";
const resolver = new ethers.Contract(resolverAddress, resolverAbi, provider);
const addresses = ['0xd8da6bf26964af9d7eed9e03e53415d37aa96045', '0x983110309620d911731ac0932219af06091b6744']
const ensAddresses = resolver.getNames(addresses);
```

This can be further expanded by using the avatar text record as a user/profile image which you can get from the ENS Name's record. Other profile information like Twitter Handle, Github Handle etc can be shown appropriately too.

This can be shown to the user as follows,

![User Pill](./images/userpill.png "User Pill")

Once the user is signed-in and the cookie is set. On return visits the cookie should be first revalidated, and if valid auto sign-in the user.

## Backwards Compatibility

Wallet providers like Torus support Sign-in via Web2 social providers linking them to an Ethereum Address. While this may be custodial, this enables new users to use Decentrliazed Identities without having to install any additional wallet extensions or applications.

This implementation can be used in parallel with existing social login providers on any website.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
