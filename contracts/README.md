# Understanding of code base

## Red Bank
The [Red Bank](./mars-red-bank/src/contract.rs) 

### States:
  - CONFIG: Item<Config>
  - GLOBAL_STATE: Item<GlobalState>
  - USERS: Map<&Addr, User>
  - MARKETS: Map<&[u8], Market>
  - MARKET_REFERENCES_BY_INDEX: Map<U32Key, Vec<u8>>
  - MARKET_REFERENCES_BY_MA_TOKEN: Map<&Addr, Vec<u8>>
  - DEBTS: Map<(&[u8], &Addr), Debt>
  - UNCOLLATERALIZED_LOAN_LIMITS: Map<(&[u8], &Addr), Uint128>

### Functions and associated steps:
- ExecuteMsg
  - execute_receive_cw20(deps, env, info, cw20_msg)
  - execute_update_config(deps, env, info, config)
  - [execute_init_asset](./mars-red-bank/src/contract.rs#L349)(deps, env, info, asset, asset_params, asset_symbol):
    1. If market dies not exist for the asset, create_market and save, else return error
    2. Set asset symbol
  - create_market: params { initial_borrow_rate, max_loan_to_value, reserve_factor, liquidation_threshold, liquidation_bonus, interest_rate_model_params, active, deposit_enabled, borrow_enabled }
  - execute_init_asset_token_callback(deps, env, info, reference)
  - execute_update_asset(deps, env, info, asset, asset_params)
  - [execute_update_uncollateralized_loan_limit](./mars-red-bank/src/contract.rs#L628)(deps, env, info, user_addr, asset, new_limit)
    - Check that the user has no collateralized debt
  - [execute_deposit](./mars-red-bank/src/contract.rs#L692)(deps, env, info, depositor_address, on_behalf_of, denom.as_bytes(), denom.as_str(), deposit_amount,)
  - execute_withdraw(deps, env, info, asset, amount, recipient_address)
  - [execute_borrow](./mars-red-bank/src/contract.rs#L964)(deps, env, info, asset, amount, recipient_address): Add debt for the borrower and send the borrowed funds. If asset is a Terra native token, the amount sent is selected so that the sum of the transfered amount plus the stability tax payed is equal to the borrowed amount.
    Borrow {
    1. Check if uncollateralized_loan_limit is reached, if not, add the debt to the borrower's debt list, with price from oracle,
    2. borrow_amount_in_uusd = borrow_amount * borrow_asset_price
    3. Uncollateralized loan: check borrow amount plus debt does not exceed uncollateralized loan limit
  - execute_repay( deps, env, info, repayer_address, on_behalf_of, denom.as_bytes(), denom.clone(), repay_amount, AssetType::Native,)
  - execute_liquidate( deps, env, info, sender, collateral_asset, Asset::Native {     denom: debt_asset_denom, }, user_addr, sent_debt_asset_amount, receive_ma_token,)
  - execute_update_asset_collateral_status(deps, env, info, asset, enable)

## interest_rates
The [interest_rates](./mars-red-bank/src/interest_rates.rs) 

Functions and associated steps:

- [apply_accumulated_interests](./mars-red-bank/src/interest_rates.rs#L32): Calculates accumulated interest for the time between last time market index was updated and current block. Applies desired side effects: 1. Updates market borrow and liquidity indices. 2. If there are any protocol rewards, builds a mint to the rewards collector and adds it to the returned response. 
    1. Update market indices: borrow_rate, liquidity_rate, borrow_index, liquidity_index
    2. Compute accrued protocol rewards
    3. accrued_protocol_rewards = borrow_interest_accrued * market.reserve_factor 
    4. if accrued_protocol_rewards > 0, then mint accrued_protocol_rewards scaled with market liquidity_index

- [update_interest_rates](./mars-red-bank/src/interest_rates.rs#L283): Update interest rates for current liquidity and debt levels. 
    1. available_liquidity = contract_current_balance - liquidity_taken
    2. liquidity_and_debt = available_liquidity + total_debt
    3. current_utilization_rate = total_debt / liquidity_and_debt
    4. update_market_interest_rates_with_model(`env`, `market`, current_utilization_rate)

## Mars protocol rewards collector
The [interest_rates](./mars-protocol-rewards-collector) 

Functions and associated steps:

- [execute_distribute_protocol_rewards](./mars-protocol-rewards-collector/src/contract.rs#L222): Send accumulated asset rewards to protocol contracts 

- [execute_swap_asset_to_uusd](./mars-protocol-rewards-collector/src/contract.rs#L327): Swap any asset on the contract to uusd
