# Exchange Concepts

The `x/exchange` module facilitates the trading of on-chain assets.

Markets provide fee structures and are responsible for identifying and triggering settlements.
Orders are created by users to indicate a desire to trade on-chain funds in a market.
The exchange module defines a portion of market fees to be paid to the chain (distributed like gas fees).

---
<!-- TOC -->
  - [Markets](#markets)
    - [Required Attributes](#required-attributes)
    - [Market Permissions](#market-permissions)
    - [Settlement](#settlement)
    - [Commitment Settlement](#commitment-settlement)
    - [Transfer Agent](#transfer-agent)
  - [Orders](#orders)
    - [Ask Orders](#ask-orders)
    - [Bid Orders](#bid-orders)
    - [Partial Orders](#partial-orders)
    - [External IDs](#external-ids)
  - [Commitments](#commitments)
  - [Payments](#payments)
  - [Fees](#fees)
    - [Order Creation Fees](#order-creation-fees)
    - [Settlement Flat Fees](#settlement-flat-fees)
    - [Settlement Ratio Fees](#settlement-ratio-fees)
    - [Commitment Fees](#commitment-fees)
    - [Exchange Fees for Orders](#exchange-fees-for-orders)
    - [Exchange Fees for Commitments](#exchange-fees-for-commitments)
    - [Exchange Fees for Payments](#exchange-fees-for-payments)


## Markets

A market is a combination of on-chain setup and off-chain processes.
They are created by a governance proposal using the [MsgGovCreateMarketRequest](03_messages.md#msggovcreatemarketrequest) message.
Most aspects of the market are then manageable using permissioned endpoints.
Fees can only be managed with a governance proposal using the [MsgGovManageFeesRequest](03_messages.md#msggovmanagefeesrequest) message.

Each market has a set of optional details designed for human-use, e.g. name, description, website url.

If a market accepts orders, it is responsible (off-chain) for identifying order matches and triggering (on-chain) settlement.
If a market accepts commitments, it is able to manage committed funds and is responsible for releasing commitments as needed.

A market receives fees for order creation and order settlement. It also defines what fees are required and what is acceptable as payments.

A market can delegate various [permissions](#market-permissions) to other accounts, allowing those accounts to use specific endpoints on behalf of the market.

Markets can restrict who can create orders or commitments with them by defining account attributes that are required to create orders. See [Required Attributes](#required-attributes).

Markets can control whether user-settlement is allowed.
When user-settlement is allowed, the [FillBids](03_messages.md#fillbids) and [FillAsks](03_messages.md#fillasks) endpoints can be used for orders in the market.

A market can also control whether orders can be created for it.
When order creation is not allowed, any existing orders can still be settled or cancelled, but no new ones can be made (in that market).

A market can separately control whether funds can be committed to it.
When commitments are not allowed, any existing commitments can still be settled or cancelled, but no new funds can be committed (to that market).

The fees collected by a market are kept in the market's account, and can be accessed using the [MarketWithdraw](03_messages.md#marketwithdraw) endpoint.

See also: [Market](03_messages.md#market).


### Required Attributes

There is a separate list of attributes required to create each order type and commitments.
If one or more attributes are required for an action, the associated account (e.g. buyer, seller, or committer) must have all of them on their account.

Required attributes can have a wildcard at the start to indicate that any attribute with the designated base and one (or more) level(s) is applicable.
The only place a wildcard `*` is allowed is at the start of the string and must be immediately followed by a period.
For example, a required attribute of `*.kyc.pb` would match an account attribute of `buyer.kyc.pb` or `special.seller.kyc.pb`, but not `buyer.xkyc.pb` (wrong base) or `kyc.pb` (no extra level).

Attributes are defined using the [x/name](/x/name/spec/README.md) module, and are managed on accounts using the [x/attributes](/x/attribute/spec/README.md) module.


### Market Permissions

The different available permissions are defined by the [Permission](03_messages.md#permission) proto enum message.

Each market manages its own set of [AccessGrants](03_messages.md#accessgrant), which confer specific permissions to specific addresses.

* `PERMISSION_UNSPECIFIED`: it is an error to try to use this permission for anything.
* `PERMISSION_SETTLE`: accounts with this permission can use the [MarketSettle](03_messages.md#marketsettle) endpoint for a market.
* `PERMISSION_SET_IDS`: accounts with this permission can use the [MarketSetOrderExternalID](03_messages.md#marketsetorderexternalid) endpoint for a market.
* `PERMISSION_CANCEL`: accounts with this permission can use the [CancelOrder](03_messages.md#cancelorder),[MarketReleaseCommitments](03_messages.md#marketreleasecommitments) and [MarketTransferCommitment](03_messages.md#markettransfercommitment) endpoints to cancel orders and release commitments in a market.
* `PERMISSION_WITHDRAW`: accounts with this permission can use the [MarketWithdraw](03_messages.md#marketwithdraw) endpoint for a market.
* `PERMISSION_UPDATE`: accounts with this permission can use the [MarketUpdateDetails](03_messages.md#marketupdatedetails), [MarketUpdateAcceptingOrders](03_messages.md#marketupdateacceptingorders), [MarketUpdateUserSettle](03_messages.md#marketupdateusersettle), [MarketUpdateAcceptingCommitments](03_messages.md#marketupdateacceptingcommitments), and [MarketUpdateIntermediaryDenom](03_messages.md#marketupdateintermediarydenom) endpoints for a market.
* `PERMISSION_PERMISSIONS`: accounts with this permission can use the [MarketManagePermissions](03_messages.md#marketmanagepermissions) endpoint for a market.
* `PERMISSION_ATTRIBUTES`: accounts with this permission can use the [MarketManageReqAttrs](03_messages.md#marketmanagereqattrs) endpoint for a market.


### Settlement

Each market is responsible for the settlement of its orders.
To do this, it must first identify a matching set of asks and bids.
The [MarketSettle](03_messages.md#marketsettle) endpoint is then used to settle and clear orders.
If the market allows, users can also settlement orders with their own funds using the [FillBids](03_messages.md#fillbids) or [FillAsks](03_messages.md#fillasks) endpoints.

During settlement, at most one order can be partially filled, and it must be the last order in its list (in [MsgMarketSettleRequest](03_messages.md#msgmarketsettlerequest)).
That order must allow partial settlement (defined at order creation) and be evenly divisible (see [Partial Orders](#partial-orders)).
The market must also set the `expect_partial` field to `true` in the request.
If all of the orders are being filled in full, the `expect_partial` field must be `false`.

All orders in a settlement must have the same `assets` denoms, and also the same `price` denoms, but the fees can be anything.
The total bid price must be at least the total ask price (accounting for partial fulfillment if applicable).

During settlement:

1. The `assets` are transferred directly from the seller(s) to the buyer(s).
2. The `price` funds are transferred directly from the buyer(s) to the seller(s).
3. All settlement fees are transferred directly from the seller(s) and buyer(s) to the market.
4. The exchange's portion of the fees is transferred from the market to the chain's fee collector.

With complex settlements, it's possible that an ask order's `assets` go to a different account than the `price` funds come from, and vice versa for bid orders.

Transfers of the `assets` and `price` bypass the quarantine module since order creation can be viewed as acceptance of those funds.
No other send-restrictions are bypassed (e.g. `x/marker` or `x/sanction` module restrictions).
E.g. If an order's funds are in a sanctioned account, settlement of that order will fail since those funds cannot be removed from that account.


### Commitment Settlement

A market can move funds committed to it by using the [MarketCommitmentSettle](03_messages.md#marketcommitmentsettle) endpoint.

During commitment settlement:

1. Holds are released on the funds to be transferred.
2. The funds are transferred.
3. The funds are re-committed in the destination accounts.

The market can collect fees as part of this settlement, but is also charged extra fees for the tx.
See also: [Commitment Fees](#commitment-fees).

The funds are re-committed regardless of the market's `accepting_commitments` value.
The accounts these funds are being re-committed to also are not required to have the create-commitment required attributes.


### Transfer Agent

During a settlement, commitment settlement, or market withdrawal, the `admin` is also used as the transfer agent.
A transfer agent is used by the `x/marker` module's [Send Restrictions](../../marker/spec/12_transfers.md#send-restrictions) to help facilitate movement of restricted coins.
E.g. an `admin` with `transfer` access for a denom and `settle` permission for a market can use a settlement to transfer restricted funds to a recipient, regardless of required attributes on that denom or the attributes of the recipient.
If an `admin` does **not** have `transfer` access for a restricted denom, settlements can still succeed if the recipient has the required attributes for that denom.


## Orders

Orders are created by users that want to trade assets in a market.

When an order is created, a hold is placed on the applicable funds.
Those funds will remain in the user's account until the order is settled or cancelled.
The holds ensure that the required funds are available at settlement without the need of an intermediary holding/clearing account.
During settlement, the funds get transferred directly between the buyers and sellers, and fees are paid from the buyers and sellers directly to the market.

Orders can be cancelled by either the user or the market.

Once an order is created, it cannot be modified except in these specific ways:

1. When an order is partially filled, the amounts in it will be reduced accordingly.
2. An order's external id can be changed by the market.
3. Cancelling an order will release the held funds and delete the order.
4. Settling an order in full will delete the order.


### Ask Orders

Ask orders represent a desire to sell some specific `assets` at a minimum `price`.
When an ask order is created, a hold is placed on the `assets` being sold.
If the denom of the `seller_settlement_flat_fee` is different from the denom of the price, a hold is placed on that flat fee too.
It's possible for an ask order to be filled at a larger `price` than initially defined.

The `seller_settlement_flat_fee` is verified at the time of order creation, but only paid during settlement.

When an ask order is settled, the `assets` are transferred directly to the buyer(s) and the `price` is transferred directly from the buyer(s).
Then the seller settlement fees are transferred from the seller to the market.

During settlement, the seller settlement fee ratio with the appropriate `price` denom is applied to the price the seller is receiving.
That result is then added to the ask order's `seller_settlement_flat_fee` to get the total settlement fee to be paid for the ask order.
In this way, the seller's settlement ratio fee can be taken out of the `price` funds that the seller is receiving.
If the `seller_settlement_flat_fee` is the same denom as the price, it can come out of the `price` funds too.

Because the fees can come out of the `price` funds, it's possible (probable) that the total `price` funds that the seller ends up with, is less than their requested price.

For example, a user creates an ask order to sell `2cow` (the `assets`) and wants at least `15chicken` (the `price`).
The market finds a way to settle that order where the seller will get `16chicken`, but the seller's settlement fee will end up being `2chicken`.
During settlement, the `2cow` are transferred from the seller to the buyer, and `16chicken` are transferred from the buyer to the seller.
Then, `2chicken` are transferred from the seller to the market.
So the seller ends up with `14chicken` for their `2cow`.

See also: [AskOrder](03_messages.md#askorder).


### Bid Orders

Bid orders represent a desire to buy some specific `assets` at a specific `price`.
When a bid order is created, a hold is placed on the order's `price` and `buyer_settlement_fees`.

When a bid order is settled, the `price` is transferred directly to the seller(s) and the assets are transferred directly from the seller(s).
Then, the buyer settlement fees are transferred from the buyer to the market.

The `buyer_settlement_fees` are verified at the time of order creation, but only paid during settlement.
They are paid in addition to the `price` the buyer is paying.

See also: [BidOrder](03_messages.md#bidorder).


### Partial Orders

Both Ask orders and Bid orders can optionally allow partial fulfillment by setting the `allow_partial` field to `true` when creating the order.

When an order is partially filled, the order's same `assets:price` and `assets:settlement-fees` ratios are maintained.

Since only whole numbers are allowed, this means that:

* `<order price> * <assets filled> / <order assets>` must be a whole number.
* `<settlement fees> * <assets filled> / <order assets>` must also be a whole number.

When an ask order is partially filled, it's `price` and `seller_settlement_flat_fee` are reduced at the same rate as the assets, even if the seller is receiving a higher price than requested.
E.g. If an ask order selling `2cow` for `10chicken` is partially settled for `1cow` at a price of `6chicken`, the seller will receive the `6chicken` but the updated ask order will list that there's still `1cow` for sale for `5chicken`.

When an order is partially filled, its amounts are updated to reflect what hasn't yet been filled.

An order that allows partial fulfillment can be partially filled multiple times (as long as the numbers allow for it).

Settlement will fail if an order is being partially filled that either doesn't allow it, or cannot be evenly split at the needed `assets` amount.


### External IDs

Orders can be identified using an off-chain identifier.
These can be provided during order creation (in the `external_id` field).
They can also be set by the market after the order has been created using the [MarketSetOrderExternalID](03_messages.md#marketsetorderexternalid) endpoint.

Each external id is unique inside a market.
I.e. two orders in the same market cannot have the same external id, but two orders in different markets **can** have the same external id.
An attempt (by a user) to create an order with a duplicate external id, will fail.
An attempt (by a market) to change an order's external id to one already in use, will fail.

The external ids are optional, so it's possible that multiple orders in a market have an empty string for the external id.
Orders with external ids can be looked up using the [GetOrderByExternalID](05_queries.md#getorderbyexternalid) query (as well as the other order queries).

External ids are limited to 100 characters.


## Commitments

A Commitment allows an account to give control of some of its funds to a market.

When funds are committed to a market, they remain in the source account and a [hold](../../hold/spec/01_concepts.md#holds) is placed on them.
Committed funds are not usable by the account they are in; only the market can move them.
The funds stay in the account until the market either moves them using the [MarketCommitmentSettle](03_messages.md#marketcommitmentsettle) endpoint or cancels the commitment in part or full.
Commitments can only be cancelled by the market (or a governance proposal).

For a market to start accepting commitments, it must have either a settlement bips, or a commitment creation flat fee defined.
If a settlement bips is defined, an intermediary denom must also be defined and a NAV must exist from the intermediary denom to the chain's fee denom.

Management of the settlement bips and commitment creation fee options is part of the fee-management governance proposal.
The `accepting_commitments` flag and intermediary denom are managed using the [MarketUpdateAcceptingCommitments](03_messages.md#marketupdateacceptingcommitments) and [MarketUpdateIntermediaryDenom](03_messages.md#marketupdateintermediarydenom) endpoints.


## Payments

A payment is used to trade fund between two accounts.

The account that creates the payment is the `source`.
The account that can accept (or reject) the payment is the `target`.
A target does not need to be declared when a payment is created.
A payment's target can be changed (by the source) until it has been accepted, rejected, or cancelled.

The amount that the `source` is providing is the `source_amount`.
The amount that the `target` is providing is the `target_amount`.
Both the `source_amount` and `target_amount` are optional, but at least one must be provided.

When a payment is created, a hold is placed the `source_amount` funds (in the `source` account).
When a payment is accepted, that hold is released.
Then the `source_amount` is sent from the `source` to the `target`, and the `target_amount` is sent from the `target` to the `source`.

A payment is uniquely identified by its `source` and `external_id`.
It is up to the `source` to choose an `external_id` that they are not already using in another payment.
Once a payment has been accepted, rejected, or cancelled, its external id can be reused by the source.
Two different sources can use the same external id.

In order to accept a payment, all the details of the payment must be provided in the request.
This ensures that the `target` accepts the terms of the payment.

Creating or accepting a payment may require an extra amount to be included in the tx fees.
This amount is defined in the exchange module [Params](06_params.md).
The amount required for a specific payment can be calculated using the [PaymentFeeCalc](05_queries.md#paymentfeecalc) query.


## Fees

Markets dictate the minimum required fees. It's possible to pay more than the required fees, but not less.

A portion of the fees that a market collects are sent to the blockchain and distributed similar to gas fees.
This portion is dictated by the exchange module in its [params](06_params.md).

There are several types of fees:

* Order creation: Flat fees paid at the time that an order is created.
* Settlement flat fees: A fee paid during settlement that is the same for each order.
* Settlement ratio fees: A fee paid during settlement that is based off of the order's price.
* Commitment fees: Fees paid in relation to commitments.

For each fee type, there is a configuration for each order type.
E.g. the ask-order creation fee is configured separately from the bid-order creation fee.

Each fee type is only paid in a single denom, but a market can define multiple options for each.
E.g. if flat fee options for a specific fee are `5chicken,1cow`, users can provide **either** `5chicken` or `1cow` to fulfill that required fee.

If a market does not have any options defined for a given fee type, that fee is not required.
E.g. if the `fee_create_ask_flat` field is empty, there is no fee required to create an ask order.

All fees except the seller settlement ratio fees must be provided with order creation, and are validated at order creation.


### Order Creation Fees

This is a fee provided in the `order_creation_fee` field of the order creation `Msg`s and is collected immediately.
These are paid on top of any gas or message fees required.

Each order type has its own creation fee configuration:

* `fee_create_ask_flat`: The available `Coin` fee options for creating an ask order.
* `fee_create_bid_flat`: The available `Coin` fee options for creating a bid order.


### Settlement Flat Fees

This is a fee provided as part of an order, but is not collected until settlement.

Each order type has its own settlement flat fee configuration:

* `fee_seller_settlement_flat`: The available `Coin` fee options that are paid by the seller during settlement.
* `fee_buyer_settlement_flat`: The available `Coin` fee options that are paid by the buyer during settlement.

The ask order's `seller_settlement_flat_fee` must be at least one of the available `fee_seller_settlement_flat` options.
The bid order's `buyer_settlement_fees` must be enough to cover one of the `fee_buyer_settlement_flat` options plus one of the buyer settlement ratio fee options.


### Settlement Ratio Fees

A [FeeRatio](03_messages.md#feeratio) is a pair of `Coin`s defining a `price` to `fee` ratio.

Each order type has its own settlement ratio fee configurations:

* `fee_seller_settlement_ratios`: The available `FeeRatio` options that are applied to the `price` received.
* `fee_buyer_settlement_ratios`: The available `FeeRatio` options that are applied to the bid order's `price`.

If a market defines both buyer and seller settlement ratios, they should define ratios in each with the same set of `price` denoms.
E.g. if there's a `fee_buyer_settlement_ratios` entry of `100chicken:1cow`, there should be an entry in `fee_seller_settlement_ratios` with a price denom of `chicken` (or `fee_seller_settlement_ratios` should be empty).

If a market requires both, but there's a price denom in the `fee_buyer_settlement_ratios` that isn't in `fee_seller_settlement_ratios`, then orders with that denom in their `price` cannot be settled.
If a market requires both, but there's a price denom in the `fee_seller_settlement_ratios` that isn't in `fee_buyer_settlement_ratios`, then bid orders with that denom in their `price` cannot be created, so ask orders with that price denom will have nothing to settle with.

A `FeeRatio` can have a zero `fee` amount (but not a zero `price` amount), e.g. `1chicken:0chicken` is okay, but `0chicken:1chicken` is bad.
This allows a market to not charge a ratio fee for a specific `price` denom.

A `FeeRatio` with the same `price` and `fee` denoms must have a larger price amount than fee amount.

#### Seller Settlement Ratio Fee

A market's `fee_seller_settlement_ratios` are limited to `FeeRatio`s that have the same `price` and `fee` denom.
This ensures that the seller settlement fee can always be paid by the funds the seller is receiving.

To calculate the seller settlement ratio fee, the following formula is used: `<settlement price> * <ratio fee> / <ratio price>`.
If that is not a whole number, it is rounded up to the next whole number.

E.g. A market has `1000chicken:3chicken` in `fee_seller_settlement_ratios`.

* An order is settling for `400chicken`: `400 * 3 / 1000` = `1.2`, which is rounded up to `2chicken`.
* An order is settling for `3000chicken`: `3000 * 3 / 1000` = `9`, which doesn't need rounding, so stays at `9chicken`.

The actual amount isn't known until settlement, but a minimum can be calculated by applying the applicable ratio to an ask order's `price`.
The seller settlement ratio fee will be at least that amount, but since it gets larger slower than the price, `<ask order price> - <ratio fee based on ask order price> - <flat fee>` is the least amount the seller will end up with.

#### Buyer Settlement Ratio Fee

A market's `fee_buyer_settlement_ratios` can have `FeeRatios` with any denom pair, i.e. the `price` and `fee` do not need to be the same denom.
It can also have multiple entries with the same `price` denom or `fee` denom, but it can only have one entry for each `price` to `fee` denom pair.
E.g. a market can have `100chicken:1cow` and also `100chicken:7chicken`, `500cow:1cow`, and `5cow:1chicken`, but it couldn't also have `105chicken:2cow`.

To calculate the buyer settlement ratio fee, the following formula is used: `<bid price> * <ratio fee> / <ratio price>`.
If that is not a whole number, it is rounded up to the next whole number.

The buyer settlement ratio fee should be added to the buyer settlement flat fee and provided in the `buyer_settlement_fees` in the bid order.
The ratio and flat fees can be in any denoms allowed by the market, and do not have to be the same.


### Commitment Fees

A market can collect commitment fees at commitment creation and/or during settlement.

A market is also charged an extra Tx fee when doing a settlement.

#### Commitment Creation Fees

Similar to order creation fees, these are paid by the committer at the time of committal.
A portion of these fees are given to the exchange.
This can only be defined as a flat fee: `fee_create_commitment_flat`: The available `Coin` fee options that are paid by the committer when committing funds.
The commitment's `creation_fee` must be at least one of the available `fee_create_commitment_flat` options.

#### Commitment Settlement Fee Collection

During a commitment settlement, a market can collect fees by populating the `fees` field in the [MsgMarketCommitmentSettleRequest](03_messages.md#msgmarketcommitmentsettlerequest).
The exchange does **NOT** keep any portion of these fees.
Only fees that have been committed to the market can be collected in this way.

#### Commitment Settlement Fee Charge

An additional fee is charged (similar to a msg-based fee) for commitment settlements that is based on the total funds being moved.
The [CommitmentSettlementFeeCalc](05_queries.md#commitmentsettlementfeecalc) query can be used to find out how much the fee will be for a commitment settlement.
This fee is provided as a portion of the tx fees (in addition to any gas or other fees).

The fee is calculated as such:

1. Sum the settlement inputs.
2. Use Marker Net-Asset-Value (NAV) entries (or provided NAV entries) to convert the total into the market's intermediary denom (any amounts in the fee denom are not converted).
3. A NAV is used to convert the intermediary total to the fee denom (which is added to any fee denom amount in the inputs).
4. The market's `commitment_settlement_bips` is applied to the fee denom total.
5. That total is divided by two to get the final fee amount.

During step 2, the amount of the intermediary denom is tracked as a real number (not limited to integers).
At the end of that step, the sum is rounded up to the next integer.

Technically both steps 4 and 5 are done together by doing `amount * bips / 20,000`. If that is not an integer, the result is rounded up.

If a NAV is needed for fee calculation, but does not exist (or wasn't provided), commitment settlement will fail.
To help, NAVs can be provided as part of a commitment settlement and the marker module will be updated with any NAV info provided.
NAVs are **NOT** updated in the query, only when provided as part of a settlement.
NAVs provided to the query are still used for that query, though.

The total is divided by two because the sum of the inputs is effectively double what we usually think of as the "value of a trade".
E.g. If buying  `1candybar` for `99cusd`, we think of the value of that trade as $0.99.
Without the halving, though, we might have ended up with a total of $1.98 by converting the `1candybar` into `cusd` (hopefully `99cusd`) and adding that to the other `99cusd`.

##### Commitment Settlement Fee Charge Example

A commitment settlement is requested with the following data and market setup:

| name               | value                                                        |
|--------------------|--------------------------------------------------------------|
| Input Total        | `30apple,93banana,87cherry,300nhash`                         |
| NAVs               | `21apple:500cherry`, `44banana:77cherry`, `10cherry:31nhash` |
| Intermediary Denom | `cherry`                                                     |
| Bips               | 25                                                           |
| Fee Denom          | `nhash`                                                      |

1. Convert the input total to the intermediary denom:
   * `30apple` * `500cherry`/`21apple` = `714.285714285714cherry`
   * `93banana` * `77cherry`/`44banana` = `162.75cherry`
   * `87cherry` => `87cherry` (already intermediary denom, do not convert yet)
   * `300nhash` => `300nhash` (already in fee denom, do not convert)
   * Sum: `964.035714285714cherry` => `965cherry` (and `300nhash`).
2. Convert that to the fee denom:
   * `965cherry` * `31nhash`/`10cherry` = `2991.5nhash` => `2992nhash`
   * `300nhash` => `300nhash` (already in fee denom, do not convert)
   * Sum: `3292nhash`
3. Apply Bips:
   * `3292nhash` * `25`/`20000` = `4.115nhash` => `5nhash`

The market will need to provide an extra `5nhash` with the `Tx` fees in order to do this commitment settlement.


### Exchange Fees for Orders

A portion of the fees collected by a market for order creation and settlement, are given to the exchange.
The amount is defined using basis points in the exchange module's [Params](06_params.md#params) and can be configured differently for specific denoms.

When the market collects fees, the applicable basis points are looked up and applied to the amount being collected.
That amount is then transferred from the market's account to the chain's fee collector (similar to gas fees).

The following formula is used for each denom in the fees being collected: `<fee amount> * <basis points> / 10,000`.
If that is not a whole number, it is rounded up to the next whole number.

For example, Say the exchange has a default split of `500` (basis points), and a specific split of `100` for `rooster`.
When a market collects a fee of `1500hen,710rooster`:
There is no specific split for `hen`, so the default `500` is used for them. `1500 * 500 / 10,000` = `75hen` (a whole number, so no rounding is needed).
The specific `rooster` split of `100` is used for those: `710 * 100 / 10,000` = `7.1` which gets rounded up to `8rooster`.
So the market will first receive `1500hen,710rooster` from the buyer(s)/seller(s), then `75hen,8rooster` is transferred from the market to the fee collector.
The market is then left with `1425hen:702rooster`.

During [MarketSettle](03_messages.md#marketsettle), the math and rounding is applied to the total fee being collected (as opposed to applying it to each order's fee first, then summing that).

During order creation, the exchange's portion of the order creation fee is calculated and collected from the creation fee provided in the `Msg`.

During [FillBids](03_messages.md#fillbids) or [FillAsks](03_messages.md#fillasks), the settlement fees are summed and collected separately from the order creation fee.
That means the math and rounding is done twice, once for the total settlement fees and again for the order creation fee.
This is done so that the fees are collected the same as if an order were created and later settled by the market.


### Exchange Fees for Commitments

When a commitment is created, a portion of the fee collected by the market is given to the exchange in the same manner that order creation fees are handled.

During a commitment settlement, the exchange collects a fee proportional to the funds being settled.
This fee must be included in the `Tx` fees provided with a `MsgMarketCommitmentSettleRequest`.
See [Commitment Settlement Fee Charge](#commitment-settlement-fee-charge) for details.


### Exchange Fees for Payments

When a payment is created with a non-zero `source_amount`, an extra amount is required to be included in the tx fees.
When a payment is accepted with a non-zero `target_amount`, an extra amount is required to be included in the tx fees.

The amounts are flat and defined in the exchange module [Params](06_params.md) with separate entries for creating and accepting payments.
The [PaymentFeeCalc](05_queries.md#paymentfeecalc) query can be used to identify the extra required tx fee amounts.
