# ZIP 10: Reducing block time 

## Summary and Motivation

For users that interact with ZetaChain on the Cosmos side or EVM side the main perceived
latency comes from block time when chain is not congested.  Currently the block time is around 6s,
and as ZetaChain uses Tendermint consensus which is 1-block finality, the latency of user
transaction is around 6-12s. This perceived transaction latency can be reduced by having
a shorter block time.  There is precedence that a production Cosmos SDK based chain can achieve
~1.2s block time. 


## Mechanisms



## Impacts

### Components of `zetacore` that depends on the block time being around 6s
- Emission module: fixed amount per block

### Components of `zetaclient` that depends on the block time being around 6s
- Outbound scheduler interval--they are in unit of blocks, and calibrated for block time being 6s.
- All ticker parameters in `Chain Params` are calibrated for block time being 6s (example params below)
```
% zetacored q observer show-chain-params 42161
chain_params:
  ballot_threshold: "0.660000000000000000"
  chain_id: "42161"
  confirmation_count: "90"
  connector_contract_address: "0x0000000000000000000000000000000000000000"
  erc20_custody_contract_address: 0xECe33274237E6422f2668eD7dEE5901b16336aA0
  gas_price_ticker: "300"
  gateway_address: 0x1C53e188Bc2E471f9D4A4762CFf843d32C2C8549
  inbound_ticker: "5"
  is_supported: true
  min_observer_delegation: "10000000000000000000.000000000000000000"
  outbound_schedule_interval: "15"
  outbound_schedule_lookahead: "30"
  outbound_ticker: "8"
  watch_utxo_ticker: "0"
  zeta_token_contract_address: "0x0000000000000000000000000000000000000000"
```

### Storage growth

### Computation requirements and block gas limit

## Test cases

## Implementation

## References

Osmosis block reduction forum post: https://forum.osmosis.zone/t/lower-blocktimes-in-accordance-with-sync-speeds/2558
