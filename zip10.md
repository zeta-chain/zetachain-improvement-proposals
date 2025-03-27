---
zip: 10
title: Reducing block time
author: brewmaster012
status: Draft
created: 2025-03-20
---

## Summary and Motivation

For users that interact with ZetaChain on the Cosmos side or EVM side the main perceived
latency comes from block time when chain is not congested.  Currently the block time is around 6s,
and as ZetaChain uses Tendermint consensus which is 1-block finality, the latency of user
transaction is around 6-12s. This perceived transaction latency can be reduced by having
a shorter block time.  There is precedence that a production Cosmos SDK based chain can achieve
~1.2s block time. 


## Mechanisms

Traditionally the block time is controlled by the tendermint config files of the validators
`.zetacored/data/config/config.toml`. An example configuration is as follows currently
```
#######################################################
###         Consensus Configuration Options         ###
#######################################################
[consensus]

wal_file = "data/cs.wal/wal"

# How long we wait for a proposal block before prevoting nil
timeout_propose = "3s"
# How much timeout_propose increases with each round
timeout_propose_delta = "500ms"
# How long we wait after receiving +2/3 prevotes for “anything” (ie. not a single block or nil)
timeout_prevote = "1s"
# How much the timeout_prevote increases with each round
timeout_prevote_delta = "500ms"
# How long we wait after receiving +2/3 precommits for “anything” (ie. not a single block or nil)
timeout_precommit = "1s"
# How much the timeout_precommit increases with each round
timeout_precommit_delta = "500ms"
# How long we wait after committing a block, before starting on the new
# height (this gives us a chance to receive some more precommits, even
# though we already have +2/3).
timeout_commit = "5s"

# How many blocks to look back to check existence of the node's consensus votes before joining consensus
# When non-zero, the node will panic upon restart
# if the same consensus key was used to sign {double_sign_check_height} last blocks.
# So, validators should stop the state machine, wait for some blocks, and then restart the state machine to avoid panic.
double_sign_check_height = 0

# Make progress as soon as we have all the precommits (as if TimeoutCommit = 0)
skip_timeout_commit = false

# EmptyBlocks mode and possible interval between empty blocks
create_empty_blocks = true
create_empty_blocks_interval = "0s"

# Reactor sleep duration parameters
peer_gossip_sleep_duration = "100ms"
peer_query_maj23_sleep_duration = "2s"
```

The various timeouts control the best case block time (no new block can finalize before 5s timeout).
To reduce the block time, one mechanism is to tune this parameter; however this must be done by
all validators at the same time, otherwise the network behavior is hard to predict. 

A better idea is to move the config to on-chain state parameters so that they can be changed
by gov proposal and take effect uniformly across all validators and at upgrade time. 

Furthermore, it would be prudent to reduce block time in steps gradually and observe effects for
unknown and unforseeable issues, therefore it'll be great to include the target block time
in the state updatable by gov proposal, and parameterize across the stack (most importantly
the ticker and interval parameters in zetaclient) with target block time. 

## Impacts

### Components of `zetacore` that depends on the block time being around 6s
- Emission module: fixed amount per block
- Block Signing Window and Slashing params need to be adjusted to account for shorter blocktimes. 

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
