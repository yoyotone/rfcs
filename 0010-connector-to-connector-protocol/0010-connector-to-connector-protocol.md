---
title: The Connector to Connector protocol
draft: 2
---
# Connector to Connector protocol

## Discovery

Connectors discover their peers through out-of-band communication, or by looking at https://connector.land and contacting the administrator of another connector.

Once peered, two connectors have a ledger between them; this is often a ledger with just two accounts, often administered collaboratively by the two connectors.

There are two ways for connectors to peer with each other: with, or without WebFinger.

In WebFinger-based discovery, both peers still need some out-of-band communication channel, over which one
prospective peer tells the other:
* their intent, "Let's peer using WebFinger!"
* their own hostname
* the currency code they propose for the peer ledger
* the ledger scale they propose for the peer ledger

And the other peer response with:
* their agreement, "OK, let's peer using WebFinger for discover, and using that currency code and scale for the peer ledger!"
* their own hostname

Now, both peers look up each other's host resource, for instance:

* the server wallet1.com looks up https://wallet2.com/.well-known/webfinger?resource=https://wallet2.com
* the server wallet2.com looks up https://wallet1.com/.well-known/webfinger?resource=https://wallet1.com

This way, each peer has the other peer's public key. They now use ECDH to create a shared secret, from which a ledger prefix and an auth token are derived,
as described in JavaScript in Appendix A.

When peering without WebFinger, the first peer tells the other:
* their intent, "Let's peer without WebFinger!"
* their full RPC uri
* the protocol version the other party can use for making RPC calls
* an auth token the other party can use for making RPC calls
* the ledger prefix they propose for the peer ledger
* the currency code they propose for the peer ledger
* the ledger scale they propose for the peer ledger

And the other peer response with:
* their agreement, "yes, let's peer! Thanks for your details, here are mine!"
* their own full RPC uri
* the protocol version the first party can use for making RPC calls
* an auth token the first party can use for making RPC calls

In both cases, each peer ends up knowing:

* the protocol version to use when making RPC calls to the other peer
* the endpoint URL to use when making RPC calls to the other peer
* the ledger prefix to use when making RPC calls to the other peer
* the auth token to use when making RPC calls to the other peer
* the currency code for the peering ledger
* the currency scale for the peering ledger

## Route broadcasts

Connectors send each other `broadcast_routes` messages, using the message sending functionality of the ledger between them.

The syntax of such a method is defined by https://github.com/interledgerjs/ilp-connector/blob/v17.0.2/schemas/RoutingUpdate.json and
 https://github.com/interledgerjs/five-bells-shared/blob/v22.0.1/schemas/Routes.json.

When a route is included in a route update from Alice to Bob, Alice is stating she will, at least temporarily, be able to forward a payment to that destination, if the
source amount which Bob would send her is high enough, given the destination amount, and as described by the `points` piece-wise linear function.

Connectors also exchange quote requests,
in the same way users may request a quote from a connector, see [IL-RFC 8](../0008-interledger-quoting-protocol/0008-interledger-quoting-protocol.md).
This process is called remote quoting.

Note that the points series ("liquidity curves") in route broadcasts, as well as the
amounts in quote requests and quote responses, are expressed in integer values in the ledger's base unit of the ledger. This means you will need to divide/multiply
by `10 ** currency_scale` for the ledger in question, to convert integers (in ledger units) to floats (in terms of the ledger's announced `currency_code`), and back.

The curve from the route broadcast is used directly for determining a source amount / destination amount relation, and remote quoting is only used when
a connector knows about a route without knowing its curve information (this can happen if a connector somewhere in the network was explicitly configured with a hard-wired
(default) route in its ilp-connector config; that route then also gets forwarded without curve in route broadcasts across the network, and all connectors that want to use this
route will therefore need to use remote quoting).
All routes that were built up from local pairs will however get broadcast including their curves, based on connector fees, currency exchange rates, and liquidity limits (min and
max limits on the connector's balance on both ledgers) and this requires routes to be re-broadcast when their liquidity curve changes, but ilp-kit will aggregate over time, both
for its own liquidity changes, and route changes which it forwards, into one message per 30 seconds.

When combining various alternative (parallel) routes, for each section of the liquidity curve, ilp-kit will only consider routes which are as short as the shortest known route,
and from the set of shortest paths, choose the cheapest one. Note that the routing table of each ilp-kit is used for two things:

* determine whether a payment's source amount is sufficient or not, given its destination amount
* if so, choose the next hop (it's then up to that peer to choose the hop after that, until the destination node is reached)

## Please expand this document

This document is a stub, please help expand it! See https://github.com/interledger/rfcs.

## Appendix A: WebFinger-based peering

### dependencies
```js
const crypto = require('crypto')
const tweetnacl = require('tweetnacl')
const fetch = require('node-fetch')
const https = require('https')
```

### inputs from configuration
```js
const myHostname = 'wallet1.com'
const myRpcUriPath = '/rpc'
const httpsOptions = { ... }
```

### inputs from out-of-band communication
```js
const peerHostname = 'wallet2.com'
const ledgerCurrency = '.usd.9.'
```

### STEP 1: generate your own key pair
```js
  const myPriv = crypto.createHmac('sha256', crypto.randomBytes(33)).update('CONNECTOR_ED25519')
  const myPub = tweetnacl.scalarMult.base(crypto.createHash('sha256').update(myPriv).digest())
```

### STEP 2: host your public key in your WebFinger record
```js
function serverWebFinger(httpsOptions, myHostname, myRpcUriPath, myPub) {
  https.createServer( httpsOptions, (req, res) => {
    if (req.url.startsWith(myRcpUriPath)) {
      // handle rpc call
    } else if (req.url.startsWith('/.well-known/webfinger') {
      res.end(JSON.stringify({
          subject: 'https://' + ownHostname,
          properties: {
            'https://interledger.org/rel/protocolVersion': `Compatible: ilp-kit v3.0.0`,
            'https://interledger.org/rel/publicKey': myPub.toString('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_')
          },
          links:[
            { rel: 'https://interledger.org/rel/peersRpcUri', href: 'https://' + myHostname + myRpcUriPath }
          ]
        }
      }, null, 2))
    }
  }).listen(443)
}
```

### STEP 3: retrieve your peer's public key from their WebFinger record
```js
function(peerHostname) {
  return fetch('https://' + peerHostname + '/.well-known/webfinger?resource=https://' + peerHostname).then(response => {
    return response.json()
  }).then(webfingerRecord =>
    return Buffer.from(webfingerRecord.properties['https://interledger.org/rel/publicKey'], 'base64')
  })
}
```

### STEP4: calculate the peer ledger prefix and authorization token
```js
function getToken(input, myPriv, peerPub) {
  return crypto.createHmac('sha256', tweetnacl.scalarMult(
    crypto.createHash('sha256').update(toBuffer(myPriv)).digest(),
    peerPub)).update(input, 'ascii').digest()
}

const ledgerPrefix = 'peer.' + getToken('token', myPriv, peerPub).toString('base64').substring(0, 5).replace(/\+/g, '-').replace(/\//g, '_') + ledgerCurrency
const authToken = getToken('authorization', myPriv, peerPub).toString('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_')
```
