# Kinetex xSwap

Kinetex xSwap smart contracts for audit

## Description

xSwap protocol is an aggregation protocol that allows users to perform
cross-chain swaps without a need from them to do actions manually in
non-initial chain.

The protocol defines swap structure, that describes a cross-chain swap
operation. Swap consists of steps, each of them represents:

* a contract call on a certain chain with asset input and output validation
  (usually an aggregated call to a bunch of DEX'es)
* a bunch of uses of the output assets (most common uses include: gas payment
  to a relayer, bridge contract call, asset transfer to address)

The swap data is validated and signed by user. The signature allows to call
swap operation steps by an arbitrary executor securely. The protocol also
allows manual call of a swap step by user with no signature providing needed.

There is a stealth variant of the swap, where user signs a swap structure that
contains only an array of step hashes. The actual content of a step is revealed
at the execution time.

There are two main categories of the contracts: `core` and `protocols`.

### Core

Core contracts are essential parts of the xSwap protocol.

__contracts/XSwap.sol__

Main xSwap protocol contract. Inherits `Swapper`, allows funds withdraw from
its balance by whitelisted accounts.

__contracts/core/permit/PermitResolver.sol__

Default permit resolver. Transforms permit params and signature to allowance
of caller according to the EIP-2612 standard.

__contracts/core/permit/DaiPermitResolver.sol__

`PermitResolver`-compatible contract for DAI token implementation of permit.

__contracts/core/permit/SignatureDecomposer.sol__

Helper contract for work with signature components in permit resolvers.

__contracts/core/asset/NativeReceiver.sol__

Helper contract inheriting which allows contract to receive network's native
coin.

__contracts/core/asset/NativeReturnMods.sol__

Helper contract which adds a modifier to return network's native coin
unclaimed by a `NativeClaimer` on method exit.

__contracts/core/asset/TokenHelper.sol__

Helpers that unify interaction with assets (tokens/native): transfer, balance
check, claim/approve/revoke by contract.

__contracts/core/asset/NativeClaimer.sol__

Helper to count how much native funds received via `msg.value` were consumed.
Unconsumed funds are returned with `NativeReturnMods`'s modifier.

__contracts/core/asset/TokenChecker.sol__

Helpers to check token limits like min/max amount.

__contracts/core/delegate/Delegate.sol__

`Delegate` is a mini-contract that can receive token/native assets and provides
access to them for owner and creator (i.e. `DelegateManager`). Each delegate is
individual per user. Address of a delegate contract is deterministic and can be
predicted prior the deployment of it. To reduce size of contract, it's deployed
as minimal proxy (`EIP-1167`) of the original delegate contract.

__contracts/core/delegate/DelegateManager.sol__

Delegate contract manager. Responsible for deterministic deployment of the
`Delegate` contracts and providing withdraw access for whitelisted accounts.
The whitelist is expected to contain the `XSwap` main contract only.

__contracts/core/misc/LifeControl.sol__

Contract that implements logic of controlling contract's life state: pausing,
unpausing, and termination. The controlled contract holds the address of
the controller and ensures value of the `paused()` is `false` before
executing an action.

__contracts/core/misc/AccountCounter.sol__

In-memory mapping of account (`address`) to count (`uint256`). Has pre-defined
max size & uses array with `O(N)` search as backend. Besides get/set methods,
provides additional math helpers for adding/subtracting counts.

__contracts/core/misc/SimpleInitializable.sol__

Abstract contract that supports initialization as an extra step. Helpful
when contract constructor cannot be invoked (example - minimal proxy).
Protected from double-initialization. Remembers initializer account address.

__contracts/core/whitelist/AccountWhitelist.sol__

Owned list of account addresses. Accounts can be added/removed by owner.
An arbitrary account can be checked for existence in the list. Provides
method for getting list of all whitelisted accounts.

__contracts/core/withdraw/Withdrawable.sol__

Abstract contract that provides token/native withdraw functionality from its
address. The permission logic is defined by contract that inherits it.

__contracts/core/withdraw/WhitelistWithdrawable.sol__

Abstract contract that provides `Withdrawable` functionality with a check
if `msg.sender` is in a special account whitelist prior.

__contracts/core/swap/Swapper.sol__

Contract responsible for swapping flow. Includes parameters validation, the
on-chain call with output validation, and use protocol calls.

__contracts/core/swap/Swap.sol__

Defines swap-related data structures shared across multiple files.

__contracts/core/swap/SwapSignatureValidator.sol__

Contract responsible for swap signature validation. Performs swap data hashing
according to `EIP-712` and checks provided signature for validity against
computed hash.

### Protocols

Contracts built for use by the xSwap protocol in specific use-cases.

__contracts/protocols/Transfer.sol__

Simple asset transfer protocol. Transfers specified asset to the account in the
current network. Exactly one input & one output with all field content matching.
No extra args.

__contracts/protocols/bridges/CBridgeV2.sol__

Bridge hop wrapper protocol for cBridge. Exactly one input & one output.
The slippage value is calculated from output min/max. The account param
serves as receiver in destination network specified by the chain. In v2
an ability to override min/max slippage deduction via args was introduced.

__contracts/protocols/bridges/Hyphen.sol__

Bridge hop wrapper for Hyphen. Exactly one input & one output. The slippage
value is calculated from output min/max. The account serves as receiver in
destination network specified by the chain. No extra args

__contracts/protocols/gas/IGasVendor.sol__

Interface that must be implemented by an automation gas vendor protocol.

__contracts/protocols/gas/vendors/XRelayGasVendor.sol__

`IGasVendor` implementation that is compatible with `XRelay` contract.
The contract injects the following fee info to message data: collector
address to send native coin to and minimum amount to send.

__contracts/protocols/gas/GasVendorV2.sol__

Vendor-based gas payment protocol. Bound to one `IGasVendor`-compatible
contract. Gets fee details from the vendor, validates amount, and sends it
to the fee collector. Accepts one input and one dummy output (see explanation
below). The caller must match specified account. Chain ID must match current
chain. No extra args.

The v2 was introduced to fix an issue w/ some wallets failing to sign typed
data if a value that is being signed contains an empty array of structures.
The previous version had zero outputs, now we add one dummy output.

## Development

This project uses the following stack:

- Language: Solidity v0.8.16
- Framework: Hardhat
- Node.js: v18
- Yarn: v1.22

### Setup Environment

1. Ensure you have relevant Node.js version. For NVM: `nvm use`

2. Install dependencies: `yarn install`

3. Setup environment variables:

    * Clone variables example file: `cp .env.example .env`
    * Edit `.env` according to your needs

### Commands

Below is the list of commands executed via `yarn` with their descriptions:

 Command                | Alias            | Description
------------------------|------------------|------------
 `yarn hardhat`         | `yarn h`         | Call [`hardhat`](https://hardhat.org/) CLI
 `yarn build`           | `yarn b`         | Compile contracts (puts to `artifacts` folder)

## Licensing

The primary license for Kinetex xSwap is the Business Source License 1.1 (`BUSL-1.1`), see [`LICENSE`](./LICENSE).
However, some files are dual licensed under `GPL-2.0-or-later`:

- Several files in `contracts/` may also be licensed under `GPL-2.0-or-later` (as indicated in their SPDX headers),
  see [`LICENSE_GPL2`](./LICENSE_GPL2)

### Other Exceptions

- All `@openzeppelin` library files are licensed under `MIT` (as indicated in its SPDX header),
  see [`LICENSE_MIT`](./LICENSE_MIT)
