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

To achieve mentioned cross-chain automation, bridged funds that require
additional manipulations in other chain are received to so-called "delegates".
These mini-contracts provide access for xSwap protocol without needing approval
for each token/chain from user, while still allowing their owner to withdraw
funds manually at any time.

The next sections describe xSwap protocol contract files in more detail.
There are two main categories of the contracts: [core](#core) and
[protocols](#protocols).

### Core

Core contracts are essential parts of the xSwap protocol.

[__contracts/XSwap.sol__](contracts/XSwap.sol)

Main xSwap protocol contract. Inherits `Swapper` functionality. Allows funds
withdraw from its balance by whitelisted accounts. Defines life control logic
of the protocol.

The _constructor_ accepts `params` structure, that contains addresses of
auxiliary contracts:

* `swapSignatureValidator` - address of `SwapSignatureValidator` contract
* `permitResolverWhitelist` - address of `AccountWhitelist` contract
  containing addresses of permit resolver contracts
* `useProtocolWhitelist` - address of `AccountWhitelist` contract
  containing addresses of xSwap protocol contracts
* `delegateManager` - address of `DelegateManager` contract
* `withdrawWhitelist` - address of `AccountWhitelist` contract
  containing accounts allowed to withdraw from `XSwap`
* `lifeControl` - address of `LifeControl` contract

[__contracts/core/permit/PermitResolver.sol__](contracts/core/permit/PermitResolver.sol)

Default permit resolver. Transforms permit params and signature to allowance
of caller according to the EIP-2612 standard.

Inherits `SignatureDecomposer` to use its helper methods. Defines
`resolvePermit` method that accepts permit parameters and resolves permit
into `msg.sender` allowance.

[__contracts/core/permit/DaiPermitResolver.sol__](contracts/core/permit/DaiPermitResolver.sol)

`PermitResolver`-compatible contract for DAI token implementation of permit.

[__contracts/core/permit/UniswapPermitResolver.sol__](contracts/core/permit/UniswapPermitResolver.sol)

`PermitResolver`-compatible contract for permit via [Uniswap's Permit2](https://github.com/Uniswap/permit2).

[__contracts/core/permit/SignatureDecomposer.sol__](contracts/core/permit/SignatureDecomposer.sol)

Helper contract for work with signature components (`r`, `s`, `v`) in permit
resolvers.

[__contracts/core/asset/NativeReceiver.sol__](contracts/core/asset/NativeReceiver.sol)

Helper contract inheriting which allows the contract to receive network's
native coin.

[__contracts/core/asset/NativeReturnMods.sol__](contracts/core/asset/NativeReturnMods.sol)

Helper contract which adds `returnUnclaimedNative(claimer)` modifier to return
network's native coin unclaimed by `claimer` (`NativeClaimer`) on method exit.

[__contracts/core/asset/TokenHelper.sol__](contracts/core/asset/TokenHelper.sol)

Helpers that unify interaction with assets (tokens/native): transfer, balance
check, claim/approve/revoke by contract. The library includes:

* `NATIVE_TOKEN` - network's native coin representation of the xSwap protocol
* `isNative(token)` - returns `true` if `token` represents native coin
* `balanceOf(token, owner, claimer)` - returns `token` balance of `owner`
* `balanceOfThis(token, claimer)` - same as `balanceOf` with `owner` being
  `address(this)`
* `transferToThis(token, from, amount, claimer)` - transfers `amount` of `token`
  from `from` address to `address(this)`. Native coin can only be claimed from
  `msg.value` (the remaining claimable balance is tracked by `NativeClaimer`)
* `transferFromThis(token, to, amount)` - transfers `amount` of `token`
  from `address(this)` to `to`
* `approveOfThis(token, spender, amount)` - approves `amount` of
  `address(this)`-owned `token` to `spender`
* `revokeOfThis(token, spender)` - revokes approve of `address(this)`-owned
  `token` from `spender`

[__contracts/core/asset/NativeClaimer.sol__](contracts/core/asset/NativeClaimer.sol)

Helper to count how much native funds received via `msg.value` were consumed.
Unconsumed funds are returned with `NativeReturnMods`'s modifier. Members:

* `claimed(claimer)` - amount of `msg.value` that has been claimed
* `unclaimed(claimer)` - amount of `msg.value` that is available to claim
* `claim(claimer, amount)` - adds `amount` to the claimed `msg.value` counter.
  Ensures the amount doesn't exceed available unclaimed amount

[__contracts/core/asset/TokenChecker.sol__](contracts/core/asset/TokenChecker.sol)

Helpers to check token limits like min/max amount. Includes:

* `checkMin(check, amount)` - validates `amount` to be greater or equal
  `check.min`. Returns `check.max`-capped `amount`
* `checkMinMax(check, amount)` - validates `amount` to be greater or equal
  `check.min` and less or equal `check.max`
* `checkMinMaxToken(check, amount, token)` - validates `amount` to be greater
  or equal `check.min` and less or equal `check.max` and and `token` to be
  equal `check.token`

[__contracts/core/delegate/Delegate.sol__](contracts/core/delegate/Delegate.sol)

`Delegate` is a mini-contract that can receive token/native assets and provides
access to them for owner and creator (i.e. `DelegateManager`). Each delegate is
individual per user account. Address of a delegate contract is deterministic
and can be predicted prior the deployment of it. To reduce size of contract,
it's deployed as minimal proxy (`EIP-1167`) of the original delegate contract.

`Delegate` inherits `SimpleInitializable`, `Ownable` (OpenZeppelin),
`Withdrawable`, and `NativeReceiver`. The contract defines withdraw check
to allow be performed by its owner only. Also defines `setOwner(newOwner)`
that can only be called by the initializer.

[__contracts/core/delegate/DelegateManager.sol__](contracts/core/delegate/DelegateManager.sol)

Delegate contract manager. Responsible for deterministic deployment of the
`Delegate` contracts and providing withdraw access for whitelisted accounts.
The whitelist is expected to contain the `XSwap` main contract only.

The _constructor_ accepts addresses of auxiliary contracts:

* `delegatePrototype` - address of `Delegate` contract to deploy by cloning
* `withdrawWhitelist` - address of `AccountWhitelist` contract containing
  accounts that are allowed to withdraw from delegates

The following methods are provided by `DelegateManager`:

* `predictDelegateDeploy(account)` - returns predicted address of `Delegate`
  contract that will be deployed for given `account`
* `deployDelegate(account)` - deploys `Delegate` for `account` by cloning
  `delegatePrototype`. Setups `account` as the delegate owner
* `isDelegateDeployed(account) ` - returns `true` if delegate has been already
  deployed for the `account`
* `withdraw(account, withdraws)` - performs `withdraws` from the delegate of
  `account` if caller is in the `withdrawWhitelist`. The delegate deployment
  must be ensured prior the call

[__contracts/core/misc/LifeControl.sol__](contracts/core/misc/LifeControl.sol)

Contract that implements logic of controlling contract's life state: pausing,
unpausing, and termination. The controlled contract holds the address of
the controller and ensures value of the `paused()` is `false` before
executing an action.

Inherits `Ownable` (OpenZeppelin), `Pausable` (OpenZeppelin). Provides:

* `pause()` - transits controller to paused state (owner only)
* `unpause()` - transits controller to unpaused state (owner only)
* `terminate()` - locks controller in paused state forever (owner only)
* `terminated()` - returns `true` if the termination has been
  applied to controller

[__contracts/core/misc/AccountCounter.sol__](contracts/core/misc/AccountCounter.sol)

In-memory mapping of account (`address`) to count (`uint256`). Has pre-defined
max size and uses array of that size as backend. Besides get/set methods,
provides additional math helpers for adding/subtracting counts. List of methods:

* `create(maxSize)` - creates new instance of `AccountCounter` with max size
  limited to `maxSize`. The instance doesn't contain any elements (size is 0)
* `size(counter)` - returns number of elements in the `counter`
* `indexOf(counter, account, insert)` - searches (with `O(N)` complexity)
  for `account` element in the `counter` and returns index of found element
  (which allows `O(1)` access to the element later). If `insert` is `true`,
  the element is added if the `account` record doesn't exist. Otherwise null
  index is returned. If inserting an element exceeds `maxSize` - error is raised
* `indexOf(counter, account)` - overload of `indexOf(counter, account, insert)`
  with `insert` parameter set to `true`
* `isNullIndex(index)` - returns `true` if given `index` is null
* `accountAt(counter, index)` - returns `account` at `index` in `counter`
* `get(counter, account)` - returns count in `counter` for `account` (`O(N)`)
* `getAt(counter, index)` - returns count in `counter` for `index` (`O(1)`)
* `set(counter, account, count)` - sets count in `counter` to `count`
  for `account` (`O(N)`)
* `setAt(counter, index, count)` - sets count in `counter` to `count`
  for `index` (`O(1)`)
* `add(counter, account, count)` - increases count in `counter` by `count`
  for `account` (`O(N)`) and returns new count value
* `addAt(counter, index, count)` - increases count in `counter` by `count`
  for `index` (`O(1)`) and returns new count value
* `sub(counter, account, count)` - decreases count in `counter` by `count`
  for `account` (`O(N)`) and returns new count value
* `subAt(counter, index, count)` - decreases count in `counter` by `count`
  for `index` (`O(1)`) and returns new count value

[__contracts/core/misc/SimpleInitializable.sol__](contracts/core/misc/SimpleInitializable.sol)

Abstract contract that supports initialization as an extra step. Helpful
when contract constructor cannot be invoked (example - minimal proxy).
Protected from double-initialization. Remembers initializer account address.
Public methods:

* `initializer()` - returns address of initializer
* `initialized()` - returns `true` if `initialize()` has been called
* `initialize()` - initializes contract. Can only be called once.
  Sets `msg.sender` as initializer. Invokes a virtual method that is a
  subject to override for inheritor

Value origins:

* `0x4c943a984a6327bfee4b36cd148236ae13d07c9a3fe7f9857f4809df3e826db1` is
  `bytes32(uint256(keccak256("xSwap.v2.SimpleInitializable._initializer")) - 1)`

[__contracts/core/whitelist/AccountWhitelist.sol__](contracts/core/whitelist/AccountWhitelist.sol)

Owned list of account addresses. Accounts can be added/removed by owner.
An arbitrary account can be checked for existence in the list. Provides
method for getting list of all whitelisted accounts.

Inherits `Ownable` (OpenZeppelin), `SimpleInitializable`. Defines:

* `getWhitelistedAccounts()` - returns list of accounts in the whitelist
* `isAccountWhitelisted(account)`- returns `true` if the `account` is
  included to the whitelist
* `addAccountToWhitelist(account)` - adds `account` to the whitelist
  (owner only)
* `removeAccountFromWhitelist(account)` - removes `account` from the whitelist
  (owner only)

[__contracts/core/withdraw/Withdrawable.sol__](contracts/core/withdraw/Withdrawable.sol)

Abstract contract that provides token/native withdraw functionality from its
address. The permission logic is defined by contract that inherits it. Methods:

* `withdraw(withdraws)` - performs `withdraws` operations, i.e. for each
  `withdraw` sends specified `withdraw.amount` of `withdraw.token` to
  `withdraw.to` address from `address(this)`

[__contracts/core/withdraw/WhitelistWithdrawable.sol__](contracts/core/withdraw/WhitelistWithdrawable.sol)

Abstract contract that provides `Withdrawable` functionality with a check
if `msg.sender` is in a special account whitelist prior.

The _constructor_ accepts `withdrawWhitelist` - address of `AccountWhitelist`
contract to restrict list of allowed withdrawers to.

[__contracts/core/swap/Swapper.sol__](contracts/core/swap/Swapper.sol)

Contract responsible for swapping flow. Includes parameters validation, the
on-chain call with output validation, and use protocol calls.

The _constructor_ accepts addresses of auxiliary contracts:

* `swapSignatureValidator` - address of `SwapSignatureValidator` contract
* `permitResolverWhitelist` - address of `AccountWhitelist` contract
  containing addresses of permit resolver contracts
* `useProtocolWhitelist` - address of `AccountWhitelist` contract
  containing addresses of xSwap protocol contracts
* `delegateManager` - address of `DelegateManager` contract

The contract exposes swap method in two variants: `swap(SwapParams params)`
and `swapStealth(StealthSwapParams params)`. Both methods work similarly.
The difference is in the initial signature validation flow by the
`swapSignatureValidator` contract.

After signature is validated, `_performSwapStep` is called. It first validates
step deadline, chain, swapper contract, nonce. After that it resolves permits
with `_usePermits` proceeding with `_performCall` and `_performUses`.

The `_performCall` firstly transfers assets as defined by the swap structure.
There are two main flows (selected based on the `sponsor` value):

* claim by approve (or `msg.value` in case of native coin)
* claim from delegate contract (deploying it if necessary)

Once assets are claimed, on-chain call to a contract is performed. This
contract's goal is to provide output assets as defined by the signed swap
structure (consuming the input assets). The output assets come to the
`Swapper` contract, that validates its balance deltas.

Once output assets are received, the `_performUses` is executed. This method
calls the use protocols according to the swap structure. It validates that
swap protocols are whitelisted in `useProtocolWhitelist` contract.

[__contracts/core/swap/Swap.sol__](contracts/core/swap/Swap.sol)

Defines swap-related data structures shared across multiple files:

* `TokenCheck` - token check structure that allows to validate token address
  and min/max amounts
* `TokenUse` - specifies one use protocol call. Includes `protocol` address,
  `chain`, `account`, and `args` (interpretation depend on protocol), as well
  as list of expected inputs (`inIndices` - index references to `outs` of
  `SwapStep`) and outputs `outs`
* `SwapStep` - describes swap sub-operation on a certain `chain`. Must be
  performed by `swapper` contract. Specifies on-chain call inputs & outputs
  (`ins`, `sponsor`, `outs`) and protocol `uses`. Also includes user
  `nonce` and `deadline` for security
* `Swap` - describes swap operation. Its `account` plus first step's `chain`
  and `swapper` values are used for the signature validation
* `StealthSwap` - describes stealth variant of swap operation. Note that besides
  step hashes, it exposes `chain`, `swapper`, and `account` of the first step
  to make the signature validation possible
* `UseParams` - parameters structure of `IUseProtocol.use` method
* `IUseProtocol` - interface that each xSwap protocol must implement

[__contracts/core/swap/SwapSignatureValidator.sol__](contracts/core/swap/SwapSignatureValidator.sol)

Contract responsible for swap signature validation. Performs swap data hashing
according to `EIP-712` and checks provided signature for validity against
computed hash. Provides the following public interface:

* `validateSwapSignature(swap, swapSignature)` - validates that `swapSignature`
  is a valid signature of `swap` (`Swap` structure). Raises otherwise
* `validateStealthSwapStepSignature(swapStep, stealthSwap, stealthSwapSignature)` -
  validates that `stealthSwapSignature` is a valid signature of `stealthSwap`
  (`StealthSwap` structure) and that `swapStep` belongs `stealthSwap`. Returns
  index of `swapStep` when valid. Raises otherwise
* `findStealthSwapStepIndex(swapStep, stealthSwap)` - returns index of
  `swapStep` in `stealthSwap`. Raises if no match found

Note that `SwapSignatureValidator` contract uses custom `EIP-712` structure
hashing implementation rather than `EIP712` contract of OpenZeppelin's library.
This implementation allows to use arbitrary `chainId` and `verifyingContract` in
order to provide user an ability to sign swap structure in initial chain and
for the contract to validate it in this and other chains without abandoning
`EIP-712` usage. The `chainId` and `verifyingContract` values are validated
for each individual step inside `Swapper` contract (as `chain` and `swapper`
respectively).

Value origins:

* `0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f` is
  `keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)")`
* `0x759f8d0a6b014b7601ff701e703719d70a717971c25deb97628336c51d9e7d86` is
  `keccak256("xSwap")`
* `0xc89efdaa54c0f20c7adf612882df0950f5a951637e0307cdcb4c672f298b8bc6` is
  `keccak256("1")`
* `0xf4bc8c888fdd1a399fd19333c97474dda88f37f0019ac12d0594786b0d11e9f4` is
  `keccak256("Swap(address account,SwapStep[] steps)SwapStep(uint256 chain,address swapper,address sponsor,address executor,uint256 nonce,uint256 deadline,TokenCheck[] ins,TokenCheck[] outs,TokenUse[] uses)TokenCheck(address token,uint256 minAmount,uint256 maxAmount)TokenUse(address protocol,uint256 chain,address account,uint256[] inIndices,TokenCheck[] outs,bytes args)")`
* `0x725059b6e8d0caeb5f3e12ecb3d05d2f1964c25eae2e6658077a7a82d15347a1` is
  `keccak256("SwapStep(uint256 chain,address swapper,address sponsor,address executor,uint256 nonce,uint256 deadline,TokenCheck[] ins,TokenCheck[] outs,TokenUse[] uses)TokenCheck(address token,uint256 minAmount,uint256 maxAmount)TokenUse(address protocol,uint256 chain,address account,uint256[] inIndices,TokenCheck[] outs,bytes args)")`
* `0x382391664c9ae06333b02668b6d763ab547bd70c71636e236fdafaacf1e55bdd` is
  `keccak256("TokenCheck(address token,uint256 minAmount,uint256 maxAmount)")`
* `0x192f17c5e66907915b200bca0d866184770ff7faf25a0b4ccd2ef26ebd21725a` is
  `keccak256("TokenUse(address protocol,uint256 chain,address account,uint256[] inIndices,TokenCheck[] outs,bytes args)TokenCheck(address token,uint256 minAmount,uint256 maxAmount)")`
* `0x0f2b1c8dae54aa1b96d626d678ec60a7c6d113b80ccaf635737a6f003d1cbaf5` is
  `keccak256("StealthSwap(uint256 chain,address swapper,address account,bytes32[] stepHashes)")`

### Protocols

Contracts built for use by the xSwap protocol in specific use-cases.
Each protocol implements `IUseProtocol` interface defined in
`contracts/core/swap/Swap.sol`.

[__contracts/protocols/Transfer.sol__](contracts/protocols/Transfer.sol)

Simple asset transfer protocol. Transfers specified asset to the account in the
current network. Exactly one input & one output with all field content matching.
No extra args.

[__contracts/protocols/bridges/CBridgeV2.sol__](contracts/protocols/bridges/CBridgeV2.sol)

Bridge hop wrapper protocol for cBridge. Exactly one input & one output.
The slippage value is calculated from output min/max. The account param
serves as receiver in destination network specified by the chain. In v2
an ability to override min/max slippage deduction via args was introduced.

The _constructor_ accepts `cBridge` contract address and `withdrawWhitelist`.
The withdraw whitelist allows withdraw by certain accounts from the `CBridgeV2`
contract.

[__contracts/protocols/bridges/Hyphen.sol__](contracts/protocols/bridges/Hyphen.sol)

Bridge hop wrapper for Hyphen. Exactly one input & one output. The slippage
value is calculated from output min/max. The account serves as receiver in
destination network specified by the chain. No extra args.

The _constructor_ accepts `hyphen` contract address and `withdrawWhitelist`.
The withdraw whitelist allows withdraw by certain accounts from the `Hyphen`
contract.

[__contracts/protocols/gas/IGasVendor.sol__](contracts/protocols/gas/IGasVendor.sol)

Defines interface that must be implemented by an automation gas vendor protocol.

[__contracts/protocols/gas/vendors/XRelayGasVendor.sol__](contracts/protocols/gas/vendors/XRelayGasVendor.sol)

`IGasVendor` implementation that is compatible with `XRelay` contract.
The contract injects the following fee info to message data (encoded with
`abi.encode`):

* `collector` address to send native coin to (`address` type)
* minimum `amount` to send to collector (`uint256` type)

[__contracts/protocols/gas/GasVendorV2.sol__](contracts/protocols/gas/GasVendorV2.sol)

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
