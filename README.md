## Missing Maximum Asset Validation Enables DoS Attack#128



## Summary
missing thresholds for asset vector checks which brings advantage to malicious actors.

## Finding Description
The flash loan module lacks a maximum limit on the number of assets that can be borrowed in a single transaction, enabling attackers to cause denial-of-service through excessive gas consumption with high priority gas fees, since validators on Aptos prefer high priority gas fee transactions

## Impact Explanation
Thepublic fun flash_loan accepts a vector of assets without any upper bound validation. When a user calls this function with thousands of assets, the contract must process each one individually through expensive operations:

```rust
public fun flash_loan(
    initiator: &signer,
    receiver_address: address,
    assets: vector<address>
)
```

The validation function processes every asset

```rust
validation_logic::validate_flashloan_complex(
    &assets, &amounts, &interest_rate_modes
);
```

An attacker can buy multiple flash loans since there is no strict maximum threshold check per transaction. The attacker has the ability to buy infinite assets which consumes excessive gas:

```rust
for (i in 0..vector::length(assets)) {
    let asset = *vector::borrow(assets, i);
    for (j in (i + 1)..vector::length(assets)) {
        let asset_j = *vector::borrow(assets, j);
        assert!(
            asset != asset_j,
            error_config::get_einconsistent_flashloan_params()
        );
    };
}
```

This function is missing a maximum assets per transaction threshold.

Impact

Block legitimate users from using flash loans by consuming all available block gas.

Create a systemic DoS where flash loan functionality becomes unusable for several days.

The protocol's core lending functionality depends on flash loans for arbitrage and liquidations. Disabling this feature damages the liquidity providers too.

## Likelihood Explanation
This attack is trivial to execute.

No special permissions required.

Can be automated with move Scripts.

malicious actor can execute this attack immediately upon main-net deployment.

## Proof of Concept

Step 1

Bob identifies 10 assets in Aave pool (USDC, USDT, APT, etc.) Bob prepares malicious script

Step 2

Build vector with 1,000 assets Set amounts = [1, 1, 1...] (1,000 times) Set modes = [0, 0, 0...] (1,000 times)

Step 3

Call flash_loan() with 1,000 assets Function performs 999,000 duplicate checks via (loop) Each asset processes state updates, transfers, events Single transaction uses ~56M gas (56% of block limit) (approx gas)

Step 4

Submit 2 attack transactions per block Use high priority gas fees Validators prioritize these transactions Block becomes full → legitimate flash loans fail

Step 5

Run script continuously for 24 hours Result: Complete flash loan DoS

Recommendation
Add a maximum asset limit at the start of validate_flashloan_complex for example 10, 20 or 128 , it all depend on protocol rules

```rust
public fun validate_flashloan_complex(
    assets: &vector<address>, 
    amounts: &vector<u256>, 
    interest_rate_modes: &vector<u8>
) {
    // Add this check
    assert!(
        vector::length(assets) <= 10,
        error_config::get_etoo_many_flashloan_assets()
    );
    
    // Rest of validation...
}
```
