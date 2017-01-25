# Interledger Connector To Connector Protocol

The Interledger Connector To Connector Protocol (IC2CP) is used to broadcast the reachability of ledger-destinations through manually-established relationships between connectors, over a common ledger.  The basic requirements are analogous to those for IP routing among [tier 1, 2 and 3 networks](https://en.wikipedia.org/wiki/Tier_1_network), for which BGP-4 [RFC 4271](https://tools.ietf.org/html/rfc4271) is the venerable choice.  

## Background and Terminology

Border Gateway Protocol (BGP) performs [path-vector routing](https://en.wikipedia.org/wiki/Path_vector_protocol), which trades off full knowledge of network topology (vs. [link state](https://en.wikipedia.org/wiki/Link-state_routing_protocol) for scalability.  IC2CP takes the same approach, and suffers from the same incomplete knowledge of the network.  In the Interledger case, this is arguably a more costly tradeoff.  Work continues on mitigating this, without putting unacceptable limits on scalability.

### Update Message Format

| Field    | Type            | Description |
|:---------|:----------------|:------------|
| `method` | String          | `"broadcast_routes"` |
| `data`   | RoutingUpdate   | An Object containing new routes and broken links. |

#### RoutingUpdate

| Field | Type | Description |
|:------|:-----|:------------|
| `hold_down_time` | PositiveDuration | Time in milliseconds for which the sending connector claims its routes to be fresh, without another heartbeat |
| `new_routes` | [ Route... ] | A list of Routes that have been added to the sending connector's table since the last update it has sent you |
| `unreachable_through_me` | [ Ledger... ] | A list of ledgers that have become unreachable through the sending connector |

#### Route
| Field | Type | Description |
|:------|:-----|:------------|
| `source_ledger` | IlpAddress | Starting ledger (from the sender of the route's POV) |
| `destination_ledger` | IlpAddress | Ledger being advertised as reachable through the sender |
| `min_message_window` | PositiveDuration | The minimum difference (in seconds) between the source and destination transfers' expiries |
| `source_account` | IlpAddress | The sending connector's account on source_ledger |
| `destination_account` | IlpAddress | The connector's account on destination_ledger; (this property is only for local routes) |
| `destination_precision` | PositiveInteger | The precision of the destination ledger |
| `destination_scale` | PositiveInteger | The scale of the destination ledger |
| `points` | LiquidityCurve | A list of points approximating the exchange rate curve |
