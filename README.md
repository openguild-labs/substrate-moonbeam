# Moonbeam Architecture by OpenGuild

# ![Moonbeam](media/moonbeam_architecture.png)

## What Is This?

Moonbeam is a Substrate-based blockchain that provides a full Ethereum-like environment. Its runtime architecture is designed to be compatible with Ethereum while leveraging the flexibility and upgradability of Substrate. This step-by-step tutorial will breakdown Moonbeam's architecture and pallet implementation.

## Runtime Overview

Moonbeam's runtime is built using Substrate's FRAME, which allows for a modular and upgradable blockchain architecture. The runtime is composed of various components called "pallets," each responsible for specific functionality. Key components of Moonbeam's runtime architecture are as follows.

#### Polkadot SDK Pallets

Provides the core blockchain functionality, including networking, consensus, and basic runtime operations.

From Polkadot-SDK:

- _Utility_: Allows users to use derivative accounts, and batch calls
- _Balances_: Tracks GLMR token balances
- _Sudo_: Allows a privileged account to make arbitrary runtime changes. This will be removed before launch.
- _Timestamp_: On-Chain notion of time
- _Transaction Payment_: Transaction payment (fee) management
- _Randomness Collective Flip_: A (mock) onchain randomness beacon, which will be replaced by parachain randomness by mainnet.
- _ParachainUpgrade_: A helper to perform runtime upgrades on parachains
- _ParachainInfo_: A place to store parachain-relevant constants like parachain id

#### Frontier Pallets

A set of modules that enable Ethereum compatibility, including EVM execution and RPC compatibility.

- _EVM Chain Id_: A place to store the chain id for each Moonbeam network
- _EVM_: Encapsulates execution logic for an Ethereum Virtual Machine
- _Ethereum_: Ethereum-style data encoding and access for the EVM.

#### Custom Pallets

Moonbeam-specific pallets that implement unique features and governance mechanisms. These pallets are stored in `pallets/`. They are designed for Moonbeam's specific requirements:

- _Author Inherent_: Allows block authors to include their identity in a block via an inherent.
- _Parachain Staking_: Minimal staking pallet that selects collators by total amount at stake

## Core Runtime Architecture

`runtime/moonbeam/src/lib.rs` defines the Moonbeam runtime and implements the necessary pallets. This file handles the fundamental blockchain operations. Now let's take a look at how specific pallets are implemented.

### Polkadot SDK Pallets

These pallets are core components provided by the Polkadot SDK, enabling fundamental blockchain functionality.

#### Utility

The Utility pallet in the Moonbeam runtime architecture serves as a framework for bundling and executing multiple calls in a single transaction.

```rust
impl pallet_utility::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type RuntimeCall = RuntimeCall;
	type PalletsOrigin = OriginCaller;
	type WeightInfo = moonbeam_weights::pallet_utility::WeightInfo<Runtime>;
}
```

This pallet is configured to use the runtime's custom event system (`RuntimeEvent`) for emitting events. It can make calls to other parts of the runtime using the `RuntimeCall` type, allowing it to interact with various runtime functions. The `PalletsOrigin` is set to `OriginCaller`, which defines the origin (`sender`) of utility calls. Weight calculations for the pallet's operations are handled by a custom implementation (`moonbeam_weights::pallet_utility::WeightInfo`), ensuring accurate fee estimation for utility functions. This configuration enables the Utility pallet to efficiently manage complex transactions involving multiple calls within the Moonbeam ecosystem.

#### Balances

Tracks the balances of the native GLMR token, including minting, burning, and transferring tokens.

```rust
impl pallet_balances::Config for Runtime {
	type MaxReserves = ConstU32<50>;
	type ReserveIdentifier = [u8; 4];
	type MaxLocks = ConstU32<50>;
	/// The type for recording an account's balance.
	type Balance = Balance;
	/// The ubiquitous event type.
	type RuntimeEvent = RuntimeEvent;
	type DustRemoval = ();
	type ExistentialDeposit = ExistentialDeposit;
	type AccountStore = System;
	type FreezeIdentifier = ();
	type MaxFreezes = ConstU32<0>;
	type RuntimeHoldReason = RuntimeHoldReason;
	type RuntimeFreezeReason = RuntimeFreezeReason;
	type WeightInfo = moonbeam_weights::pallet_balances::WeightInfo<Runtime>;
}
```

Balances sets different `ExistentialDeposit` values based on whether runtime benchmarks are enabled (0 for normal operation, 1 for benchmarks). It allows up to 50 reserves and 50 locks per account (`MaxReserves` and `MaxLocks`). The `Balance` type is used for account balances, and `RuntimeEvent` for emitting events. The pallet doesn't implement custom dust removal (`DustRemoval = ()`) and account data is stored in the System pallet (`AccountStore = System`). Freezing is disabled (`MaxFreezes = ConstU32<0>`), but custom `RuntimeHoldReason` and `RuntimeFreezeReason` types are defined. This configuration tailors the Balances pallet to Moonbeam's needs, enabling efficient management of account balances, reserves, and locks within the network.

#### Timestamp

The Timestamp pallet in the Moonbeam runtime architecture manages the on-chain time.

```rust
impl pallet_timestamp::Config for Runtime {
	/// A timestamp: milliseconds since the unix epoch.
	type Moment = u64;
	type OnTimestampSet = ();
	type MinimumPeriod = ConstU64<3000>;
	type WeightInfo = moonbeam_weights::pallet_timestamp::WeightInfo<Runtime>;
}
```

This pallet uses a `u64` type to represent time as milliseconds since the Unix epoch (`Moment = u64`). It doesn't specify any custom logic to execute when the timestamp is set (`OnTimestampSet = ()`). The `MinimumPeriod` is set to 3000 milliseconds (3 seconds), which is the minimum time required between blocks. This prevents timestamp manipulation by malicious validators and ensures a consistent block time.

#### Transaction Payment

The Transaction Payment pallet in the Moonbeam runtime architecture manages the fee calculation and payment for transactions.

```rust
impl pallet_transaction_payment::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type OnChargeTransaction = FungibleAdapter<Balances, DealWithFees<Runtime>>;
	type OperationalFeeMultiplier = ConstU8<5>;
	type WeightToFee = ConstantMultiplier<Balance, ConstU128<{ currency::WEIGHT_FEE }>>;
	type LengthToFee = LengthToFee;
	type FeeMultiplierUpdate = SlowAdjustingFeeUpdate<Runtime>;
}
```

Transaction Payment uses a custom `LengthToFee` struct to calculate fees based on transaction length, employing a polynomial function with two coefficients: one linear (based on `TRANSACTION_BYTE_FEE`) and one cubic (using `SUPPLY_FACTOR`). The `RuntimeEvent` type is used for emitting events. Transaction charges are handled by the `FungibleAdapter` using the Balances pallet and a custom `DealWithFees` type. Operational transactions have a fee multiplier of 5 (`OperationalFeeMultiplier = ConstU8<5>`). Weight-to-fee conversion uses a constant multiplier based on `WEIGHT_FEE`. This pallet also employs a `FastAdjustingFeeUpdate` mechanism to dynamically adjust fees. This enables the Transaction Payment pallet to calculate and collect fees efficiently, considering both transaction weight and length, while allowing for dynamic fee adjustments in the Moonbeam network.

#### Randomness

The Randomness pallet in the Moonbeam runtime architecture provides a source of randomness for the network.

```rust
impl pallet_randomness::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type AddressMapping = sp_runtime::traits::ConvertInto;
	type Currency = Balances;
	type BabeDataGetter = BabeDataGetter<Runtime>;
	type VrfKeyLookup = AuthorMapping;
	type Deposit = ConstU128<{ 1 * currency::GLMR * currency::SUPPLY_FACTOR }>;
	type MaxRandomWords = ConstU8<100>;
	type MinBlockDelay = ConstU32<2>;
	type MaxBlockDelay = ConstU32<2_000>;
	type BlockExpirationDelay = ConstU32<10_000>;
	type EpochExpirationDelay = ConstU64<10_000>;
	type WeightInfo = moonbeam_weights::pallet_randomness::WeightInfo<Runtime>;
}
```

This pallet uses a custom `BabeDataGetter` to retrieve epoch index and randomness from the relay chain, with special handling for benchmarking scenarios. It interfaces with the `Balances` pallet for currency operations and `AuthorMapping` for VRF key lookups. The configuration sets a deposit of 1 UNIT \* SUPPLY_FACTOR for randomness requests, limits the maximum random words to 100 per request, and defines block delays (minimum 2, maximum 2,000) for randomness fulfillment. It also sets expiration delays for blocks (10,000) and epochs (10,000). This setup allows the Randomness pallet to provide secure, on-chain randomness for various applications within the Moonbeam network, balancing between immediacy and security in randomness generation.

#### Parachain Upgrade

The parachain upgrade pallet, formally known as the Cumulus Parachain System pallet, manages crucial aspects of Moonbeam's operation as a parachain in the Polkadot ecosystem.

```rust
impl cumulus_pallet_parachain_system::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type OnSystemEvent = ();
	type SelfParaId = ParachainInfo;
	type ReservedDmpWeight = ReservedDmpWeight;
	type OutboundXcmpMessageSource = XcmpQueue;
	type XcmpMessageHandler = XcmpQueue;
	type ReservedXcmpWeight = ReservedXcmpWeight;
	type CheckAssociatedRelayNumber = EmergencyParaXcm;
	type ConsensusHook = ConsensusHookWrapperForRelayTimestamp<Runtime, ConsensusHook>;
	type DmpQueue = frame_support::traits::EnqueueWithOrigin<MessageQueue, RelayOrigin>;
	type WeightInfo = moonbeam_weights::cumulus_pallet_parachain_system::WeightInfo<Runtime>;
}
```

This pallet uses the runtime's event system (`RuntimeEvent`) and relies on `ParachainInfo` for self-identification. It handles cross-chain message passing (XCMP) using the `XcmpQueue` for both outbound messages and message handling. This reserves weights for Downward Message Passing (DMP) and XCMP to ensure resource availability. It employs `EmergencyParaXcm` for relay chain block number checks and uses a custom `ConsensusHook` wrapper for timestamp-related consensus operations. Downward messages are processed through a `MessageQueue` with `RelayOrigin`.

#### Parachain Info

```rust
impl parachain_info::Config for Runtime {}
```

The Parachain Info pallet in the Moonbeam runtime architecture serves a simple but crucial purpose:
Despite its minimal configuration (`impl parachain_info::Config for Runtime {}`), this pallet provides essential information about Moonbeam's identity as a parachain. It stores and allows access to the parachain's ID within the Polkadot ecosystem. This ID is a crucial piece of information used by other pallets and runtime components to facilitate cross-chain communication, parachain-specific operations, and interactions with the relay chain. The empty implementation suggests that Moonbeam uses the default behavior of this pallet without any custom modifications, relying on its standard functionality to maintain and provide access to the parachain's identity information.

### Frontier Pallets

#### EVM Chain ID

```rust
impl pallet_evm_chain_id::Config for Runtime {}
```

The Frontier EVM Chain ID pallet configuration (`impl pallet_evm_chain_id::Config for Runtime {}`), stores and manages the Ethereum Virtual Machine (EVM) chain ID for the Moonbeam network. This chain ID is a unique identifier that distinguishes Moonbeam from other EVM-compatible networks, ensuring that transactions signed for Moonbeam cannot be replayed on other networks and vice versa. The empty implementation indicates that Moonbeam uses the default behavior of this pallet without customization, relying on its standard functionality to maintain and provide access to the EVM chain ID, which is essential for proper transaction processing and network identification in Ethereum-compatible operations.

#### EVM Config

The Frontier EVM pallet in the Moonbeam runtime architecture is crucial for implementing Ethereum Virtual Machine (EVM) compatibility.

```rust
impl pallet_evm::Config for Runtime {
	type FeeCalculator = TransactionPaymentAsGasPrice;
	type GasWeightMapping = pallet_evm::FixedGasWeightMapping<Self>;
	type WeightPerGas = WeightPerGas;
	type BlockHashMapping = pallet_ethereum::EthereumBlockHashMapping<Self>;
	type CallOrigin = EnsureAddressRoot<AccountId>;
	type WithdrawOrigin = EnsureAddressNever<AccountId>;
	type AddressMapping = IdentityAddressMapping;
	type Currency = Balances;
	type RuntimeEvent = RuntimeEvent;
	type Runner = pallet_evm::runner::stack::Runner<Self>;
	type PrecompilesType = MoonbeamPrecompiles<Self>;
	type PrecompilesValue = PrecompilesValue;
	type ChainId = EthereumChainId;
	type OnChargeTransaction = OnChargeEVMTransaction<DealWithFees<Runtime>>;
	type BlockGasLimit = BlockGasLimit;
	type FindAuthor = FindAuthorAdapter<AuthorInherent>;
	type OnCreate = ();
	type GasLimitPovSizeRatio = GasLimitPovSizeRatio;
	type SuicideQuickClearLimit = ConstU32<0>;
	type GasLimitStorageGrowthRatio = GasLimitStorageGrowthRatio;
	type Timestamp = RelayTimestamp;
	type WeightInfo = moonbeam_weights::pallet_evm::WeightInfo<Runtime>;
}
```

`TransactionPaymentAsGasPrice` is used for fee calculation and a fixed gas-to-weight mapping. It employs `EthereumBlockHashMapping` for block hash lookups and restricts call origins to root addresses while preventing direct withdrawals. The EVM pallet leverages the Balances pallet for currency operations and a stack-based runner for EVM execution. It integrates Moonbeam-specific precompiles and uses the `EthereumChainId` for network identification. Transaction charging is handled by a custom `OnChargeEVMTransaction` implementation.

The EVM pallet also sets a block gas limit and uses `FindAuthorAdapter` for author discovery. It implements a gas limit ratio based on proof-of-validity size and storage growth. The relay chain's timestamp and Moonbeam-specific weight calculations are used here. The configuration enables Moonbeam to run Ethereum-compatible smart contracts and transactions, providing a seamless bridge between Substrate and Ethereum ecosystems.

#### Ethereum Config

The Ethereum pallet in the Moonbeam runtime architecture enables Ethereum-compatible functionality within the Substrate framework.

```rust
impl pallet_ethereum::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type StateRoot = pallet_ethereum::IntermediateStateRoot<Self>;
	type PostLogContent = PostBlockAndTxnHashes;
	type ExtraDataLength = ConstU32<30>;
}
```

Ethereum Config uses the runtime's event system (`RuntimeEvent`) for emitting Ethereum-related events. It employs `IntermediateStateRoot` for calculating state roots, which is crucial for maintaining Ethereum-compatible state transitions. The PostLogContent is set to `PostBlockAndTxnHashes`, for logging block and transaction hashes after processing. The `ExtraDataLength` is set to a constant 30 bytes, defining the maximum length of extra data that can be included in a block header. This configuration allows Moonbeam to process Ethereum transactions and smart contracts, maintain Ethereum-compatible state, and produce Ethereum-like blocks and receipts, effectively bridging the gap between Substrate and Ethereum ecosystems within the Moonbeam network.

### Moonbeam Specific Pallets

#### Author Inherent

The Author Inherent pallet in the Moonbeam runtime architecture manages the selection and verification of block authors.

```rust
impl pallet_author_inherent::Config for Runtime {
	type SlotBeacon = RelaychainDataProvider<Self>;
	type AccountLookup = MoonbeamOrbiters;
	type CanAuthor = AuthorFilter;
	type AuthorId = AccountId;
	type WeightInfo = moonbeam_weights::pallet_author_inherent::WeightInfo<Runtime>;
}
```

This pallet uses `RelaychainDataProvider<Self>` as the `SlotBeacon`, to synchronize with the Polkadot relay chain for slot timing. `MoonbeamOrbiters` is used for `AccountLookup`, suggesting a custom system for mapping between different account representations. The `CanAuthor` trait is implemented by `AuthorFilter`, which determines which accounts are eligible to author blocks. The `AuthorId` is set to `AccountId`, indicating that block authors are identified by their account IDs. This configuration enables Moonbeam to manage block authorship in a parachain context, ensuring that only eligible authors can produce blocks and that the authorship process aligns with the broader Polkadot ecosystem's timing and consensus mechanisms.

#### Parachain Staking

The Parachain Staking pallet in the Moonbeam runtime architecture manages the network's Proof-of-Stake system for selecting and rewarding collators.

```rust
impl pallet_parachain_staking::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type Currency = Balances;
	type MonetaryGovernanceOrigin = MonetaryGovernanceOrigin;
	/// Minimum round length is 2 minutes (10 * 12 second block times)
	type MinBlocksPerRound = ConstU32<10>;
	/// If a collator doesn't produce any block on this number of rounds, it is notified as inactive
	type MaxOfflineRounds = ConstU32<1>;
	/// Rounds before the collator leaving the candidates request can be executed
	type LeaveCandidatesDelay = ConstU32<{ 4 * 7 }>;
	/// Rounds before the candidate bond increase/decrease can be executed
	type CandidateBondLessDelay = ConstU32<{ 4 * 7 }>;
	/// Rounds before the delegator exit can be executed
	type LeaveDelegatorsDelay = ConstU32<{ 4 * 7 }>;
	/// Rounds before the delegator revocation can be executed
	type RevokeDelegationDelay = ConstU32<{ 4 * 7 }>;
	/// Rounds before the delegator bond increase/decrease can be executed
	type DelegationBondLessDelay = ConstU32<{ 4 * 7 }>;
	/// Rounds before the reward is paid
	type RewardPaymentDelay = ConstU32<2>;
	/// Minimum collators selected per round, default at genesis and minimum forever after
	type MinSelectedCandidates = ConstU32<8>;
	/// Maximum top delegations per candidate
	type MaxTopDelegationsPerCandidate = ConstU32<300>;
	/// Maximum bottom delegations per candidate
	type MaxBottomDelegationsPerCandidate = ConstU32<50>;
	/// Maximum delegations per delegator
	type MaxDelegationsPerDelegator = ConstU32<100>;
	/// Minimum stake required to be reserved to be a candidate
	type MinCandidateStk = ConstU128<{ 20_000 * currency::GLMR * currency::SUPPLY_FACTOR }>;
	/// Minimum stake required to be reserved to be a delegator
	type MinDelegation = ConstU128<{ 500 * currency::MILLIGLMR * currency::SUPPLY_FACTOR }>;
	type BlockAuthor = AuthorInherent;
	type OnCollatorPayout = ();
	type PayoutCollatorReward = PayoutCollatorOrOrbiterReward;
	type OnInactiveCollator = OnInactiveCollator;
	type OnNewRound = OnNewRound;
	type SlotProvider = RelayChainSlotProvider;
	type WeightInfo = moonbeam_weights::pallet_parachain_staking::WeightInfo<Runtime>;
	type MaxCandidates = ConstU32<200>;
	type SlotDuration = ConstU64<6_000>;
	type BlockTime = ConstU64<6_000>;
}
```

This pallet uses the `Balances` pallet for currency operations and allows monetary governance actions through a custom origin. It sets a minimum round length of 10 blocks (2 minutes) and considers a collator inactive after 1 offline round. Most staking actions (leaving candidacy, bond adjustments, delegator exits) have a 28-round delay (4 weeks), while reward payments have a 2-round delay. The system supports up to 200 candidates, with a minimum of 8 selected per round. Each candidate can have up to 300 top and 50 bottom delegations, while delegators can delegate to up to 100 candidates. The minimum stake is 20,000 GLMR for candidates and 500 milliGLMR for delegators, both adjusted by a supply factor.

Parachain Staking integrates custom implementations for collator rewards, inactive collator handling, and new round notifications, supporting Moonbeam's orbiter system. It uses the relay chain for slot timing (6-second slots) and Moonbeam-specific weight calculations. This configuration enables Moonbeam to operate a secure and flexible staking system tailored to its parachain needs.

## Precompiles

Precompiles refer to specialized functions or contracts that are hardcoded into the blockchain protocol rather than being written in any high-level smart contract language. These precompiles are executed natively by the Ethereum Virtual Machine (EVM) for performance-critical operations.

```rust
mod migrations;
mod precompiles;
pub use precompiles::{
	MoonbeamPrecompiles, PrecompileName, FOREIGN_ASSET_PRECOMPILE_ADDRESS_PREFIX,
};
```

Moonbeam implements custom precompiles through the `MoonbeamPrecompiles` struct, which contains a set of predefined Ethereum-compatible contracts with optimized implementations. These precompiles are accessible at specific addresses within the EVM, allowing efficient execution of common operations. The architecture includes a `PrecompileName` enum, a standardized way to reference and manage different precompiles. The `FOREIGN_ASSET_PRECOMPILE_ADDRESS_PREFIX` indicates support for interacting with assets from other chains, highlighting Moonbeam's cross-chain capabilities. By using precompiles, Moonbeam can offer enhanced performance for specific operations, extend the EVM's native functionality, and provide seamless integration with its unique features and cross-chain interactions, all while maintaining Ethereum compatibility.
