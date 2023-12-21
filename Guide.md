# Wallets on Injective

## Overview

Two types of wallets are used by end-users. They are `Metamask` and `Cosmos` native wallets.

If you already used some Ethereum native wallet before, the user experience will be the exact same as
you are already accustomed to.

## Address formats for clients

`EthAccounts` can be represented in both [Bech32](https://en.bitcoin.it/wiki/Bech32) and hex format
for Ethereum's Web3 tooling compatibility.

The Bech32 format is the default format for Cosmos-SDK queries and transactions through CLI and REST
clients. The hex format on the other hand, is the Ethereum `common.Address `representation of a Cosmos
`sdk.AccAddress`.

- Address (Bech32): `inj14au322k9munkmx5wrchz9q30juf5wjgz2cfqku`
- Address (EIP55 Hex): `0xAF79152AC5dF276D9A8e1E2E22822f9713474902`
- Compressed Public Key:
  `{"@type":"/injective.crypto.v1beta1.ethsecp256k1.PubKey","key":"ApNNebT58zlZxO2yjHiRTJ7a7ufjIzeq5HhLrbmtg9Y/"}`

Note: you can use Ethereum account private key and mnemonic directly

# Smart Contract

## Prerequisites

Before starting development of smart contract, make sure you have [rustup](https://rustup.rs/) along with a
recent `rustc` and `cargo` version installed. Currently, they(Injective team) are testing on 1.58.1+.

And you need to have the `wasm32-unknown-unknown` target installed as well.

You can check that via:

```sh
rustc --version
cargo --version
rustup target list --installed
# if wasm32 is not listed above, run this
rustup target add wasm32-unknown-unknown
# to install cargo-generate, run this
cargo install cargo-generate
```

## Start developing a smart contract with a Template

In your working directory, quickly launch your smart contract with the recommended folder structure
and build options by running the following commands:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch 1.0 --name my-first-contract
cd my-first-contract
```

This helps get you started by providing the basic boilerplate and structure for a smart contract.
In the [src/contract.rs](https://github.com/InjectiveLabs/cw-counter/blob/ea3b781447a87f052e4b8308d5c73a30481ed61f/src/contract.rs)
file you will find that the standard CosmWasm entrypoints
[instantiate()](https://github.com/InjectiveLabs/cw-counter/blob/ea3b781447a87f052e4b8308d5c73a30481ed61f/src/contract.rs#L15),
[execute()](https://github.com/InjectiveLabs/cw-counter/blob/ea3b781447a87f052e4b8308d5c73a30481ed61f/src/contract.rs#L35),
and [query()](https://github.com/InjectiveLabs/cw-counter/blob/ea3b781447a87f052e4b8308d5c73a30481ed61f/src/contract.rs#L72)
are properly exposed and hooked up.

## Creating a new repo from template

Assuming you have a recent version of rust and cargo (v1.58.1+) installed
(via [rustup](https://rustup.rs/)),
then the following should get you a new repo to start a contract:

Install [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) and cargo-run-script.
Unless you did that before, run this line now:

```sh
cargo install cargo-generate --features vendored-openssl
cargo install cargo-run-script
```

Now, use it to create your new contract.
Go to the folder in which you want to place it and run:

**Latest**

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --name PROJECT_NAME
```

For cloning minimal code repo:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --name PROJECT_NAME -d minimal=true
```

**Older Version**

Pass version as branch flag:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch <version> --name PROJECT_NAME
```

Example:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch 0.16 --name PROJECT_NAME
```

You will now have a new folder called `PROJECT_NAME` (I hope you changed that to something else)
containing a simple working contract and build system that you can customize.

## Create a Repo

After generating, you have a initialized local git repo, but no commits, and no remote.
Go to a server (eg. github) and create a new upstream repo (called `YOUR-GIT-URL` below).
Then run the following:

```sh
# this is needed to create a valid Cargo.lock file (see below)
cargo check
git branch -M main
git add .
git commit -m 'Initial Commit'
git remote add origin YOUR-GIT-URL
git push -u origin main
```

## Compiling and running tests

Now that you created your custom contract, make sure you can compile and run it before
making any changes. Go into the repository and do:

```sh
# this will produce a wasm build in ./target/wasm32-unknown-unknown/release/YOUR_NAME_HERE.wasm
cargo wasm

# this runs unit tests with helpful backtraces
cargo unit-test RUST_BACKTRACE=1

# auto-generate json schema
cargo schema
```

### Understanding the tests

The main code is in `src/contract.rs` and the unit tests there run in pure rust,
which makes them very quick to execute and give nice output on failures, especially
if you do `RUST_BACKTRACE=1 cargo unit-test`.

We consider testing critical for anything on a blockchain, and recommend to always keep
the tests up to date.

## Generating JSON Schema

While the Wasm calls (`instantiate`, `execute`, `query`) accept JSON, this is not enough
information to use it. We need to expose the schema for the expected messages to the
clients. You can generate this schema by calling `cargo schema`, which will output
4 files in `./schema`, corresponding to the 3 message types the contract accepts,
as well as the internal `State`.

These files are in standard json-schema format, which should be usable by various
client side tools, either to auto-generate codecs, or just to validate incoming
json wrt. the defined schema.

## Preparing the Wasm bytecode for production

Before we upload it to a chain, we need to ensure the smallest output size possible,
as this will be included in the body of a transaction. We also want to have a
reproducible build process, so third parties can verify that the uploaded Wasm
code did indeed come from the claimed rust code.

To solve both these issues, we have produced `rust-optimizer`, a docker image to
produce an extremely small build output in a consistent manner. The suggest way
to run it is this:

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer:0.12.6
```

Or, If you're on an arm64 machine, you should use a docker image built with arm64.

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer-arm64:0.12.6
```

We must mount the contract code to `/code`. You can use a absolute path instead
of `$(pwd)` if you don't want to `cd` to the directory first. The other two
volumes are nice for speedup. Mounting `/code/target` in particular is useful
to avoid docker overwriting your local dev files with root permissions.
Note the `/code/target` cache is unique for each contract being compiled to limit
interference, while the registry cache is global.

This is rather slow compared to local compilations, especially the first compile
of a given contract. The use of the two volume caches is very useful to speed up
following compiles of the same contract.

This produces an `artifacts` directory with a `PROJECT_NAME.wasm`, as well as
`checksums.txt`, containing the Sha256 hash of the wasm file.
The wasm file is compiled deterministically (anyone else running the same
docker on the same git commit should get the identical file with the same Sha256 hash).
It is also stripped and minimized for upload to a blockchain (we will also
gzip it in the uploading process to make it even smaller).

## Install `injectived`

In this section, you will install `injectived`, create an account programmatically,
and fund it to prepare for launching the smart contract on chain. FYI, `injectived`
is the command-line interface and daemon that connects to Injective and enables you
to interact with the Injective blockchain.

A Docker image has been prepared to make this tutorial easier. Alternatively, you
can follow the [installation instructions](https://docs.injective.network/develop/tools/injectived/install)
for injectived and run it locally.

Executing this command will make the docker container execute indefinitely.

```sh
docker run --name="injective-core-staging" \
-v=<directory_to_which_you_cloned_cw-template>/artifacts:/var/artifacts \
--entrypoint=sh public.ecr.aws/l9h3g6c6/injective-core:staging \
-c "tail -F anything"
```

Note: `directory_to_which_you_cloned_cw-template` must be an absolute path. The
absolute path can easily be found by running the pwd command from inside the
CosmWasm/cw-counter directory.

Open a new terminal and go into the Docker container to initialize the chain:

```sh
docker exec -it injective-core-staging sh
```

Let’s start by adding jq dependency, which will be needed later on:

```sh
# inside the "injective-core-staging" container
apk add jq
```

Now we can proceed to local chain initialization and add a test user called testuser
(when prompted use 12345678 as password). We will only use the test user to generate
a new private key that will later on be used to sign messages on the testnet:

```sh
# inside the "injective-core-staging" container
injectived keys add testuser
```

**OUTPUT**

```sh
- name: testuser
  type: local
  address: inj1exjcp8pkvzqzsnwkzte87fmzhfftr99kd36jat
  pubkey: '{"@type":"/injective.crypto.v1beta1.ethsecp256k1.PubKey","key":"Aqi010PsKkFe9KwA45ajvrr53vfPy+5vgc3aHWWGdW6X"}'
  mnemonic: ""

**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

wash wise evil buffalo fiction quantum planet dial grape slam title salt dry and some more words that should be here
```

Take a moment to write down the address or export it as an env variable, as you will need it to proceed:

```sh
# inside the "injective-core-staging" container
export INJ_ADDRESS= <your inj address>
```

Now you have successfully created testuser on Injective Testnet. The testuser address
should have 10000000000000000000 INJ balance.

To confirm, search for your address on the Injective Testnet Explorer to check your balance.

Alternatively, you can verify it by querying the bank balance -
https://k8s.testnet.lcd.injective.network/swagger/#/Query/AllBalances or with curl:

```sh
curl -X GET "https://k8s.testnet.lcd.injective.network/cosmos/bank/v1beta1/balances/<your_INJ_address>" -H "accept: application/json"
```

## Upload the Wasm Contract

In this section, we will upload the .wasm file that you compiled in the previous steps to the Injective Testnet.

```sh
# inside the "injective-core-staging" container, or from the contract directory if running injectived locally
yes 12345678 | injectived tx wasm store artifacts/my_first_contract.wasm \
--from=$(echo $INJ_ADDRESS) \
--chain-id="injective-888" \
--yes --fees=1000000000000000inj --gas=2000000 \
--node=https://k8s.testnet.tm.injective.network:443
```

**Output**

```sh
code: 0
codespace: ""
data: ""
events: []
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: '[]'
timestamp: ""
tx: null
txhash: 912458AA8E0D50A479C8CF0DD26196C49A65FCFBEEB67DF8A2EA22317B130E2C
```

Check your address on the [Injective Testnet Explorer](https://testnet.explorer.injective.network/),
and look for a transaction with the txhash returned from storing the code on chain.
The transaction type should be `MsgStoreCode`.

## Submit a code upload proposal to Injective mainnet

In this section, you will learn how to submit a proposal and vote for it.

Injective network participants can propose smart contracts deployments and vote in
governance to enable them. The wasmd authorization settings are by on-chain governance,
which means deployment of a contract is completely determined by governance. Because
of this, a governance proposal is the first step to uploading contracts to Injective mainnet.

Sample usage of injectived to start a governance proposal to upload code to the chain:

```sh
injectived tx wasm submit-proposal wasm-store artifacts/cw20_base.wasm
--title "Title of proposal - Upload contract" \
--description "Description of proposal" \
--instantiate-everybody true \
--deposit=1000000000000000000inj \
--run-as [inj_address] \
--gas=10000000 \
--chain-id=injective-888 \
--broadcast-mode=sync \
--yes \
--from [YOUR_KEY] \
--gas-prices=500000000inj
```

The command `injectived tx gov submit-proposal wasm-store` submits a wasm binary proposal.
The code will be deployed if the proposal is approved by governance.

Let’s go through two key flags `instantiate-everybody` and `instantiate-only-address`, which
set instantiation permissions of the uploaded code. By default, everyone can instantiate the contract.

```sh
--instantiate-everybody boolean # Everybody can instantiate a contract from the code, optional
--instantiate-only-address string # Only this address can instantiate a contract instance from the code
```

## Contract Instantiation Governance

As mentioned above, contract instantiation permissions on Mainnet depend on the flags used when
uploading the code. By default, it is set to permissionless, as we can verify on the genesis
`wasmd` Injective setup:

```sh
"wasm": {
            "codes": [],
            "contracts": [],
            "gen_msgs": [],
            "params": {
                "code_upload_access": {
                    "address": "",
                    "permission": "Everybody"
                },
                "instantiate_default_permission": "Everybody"
            },
            "sequences": []
        }
```

Unless the contract has been uploaded with the flag `--instantiate-everybody false`, everybody
can create new instances of that code.

## Contract Instantiation Proposal

```sh
 injectived tx gov submit-proposal instantiate-contract [code_id_int64] [json_encoded_init_args] --label [text] --title [text] --description [text] --run-as [address] --admin [address,optional] --amount [coins,optional] [flags]
```

```sh
Flags:
  -a, --account-number uint      The account number of the signing account (offline mode only)
      --admin string             Address of an admin
      --amount string            Coins to send to the contract during instantiation
  -b, --broadcast-mode string    Transaction broadcasting mode (sync|async|block) (default "sync")
      --deposit string           Deposit of proposal
      --description string       Description of proposal
      --dry-run                  ignore the --gas flag and perform a simulation of a transaction, but dont broadcast it (when enabled, the local Keybase is not accessible)
      --fee-account string       Fee account pays fees for the transaction instead of deducting from the signer
      --fees string              Fees to pay along with transaction; eg: 10uatom
      --from string              Name or address of private key with which to sign
      --gas string               gas limit to set per-transaction; set to "auto" to calculate sufficient gas automatically (default 200000)
      --gas-adjustment float     adjustment factor to be multiplied against the estimate returned by the tx simulation; if the gas limit is set manually this flag is ignored  (default 1)
      --gas-prices string        Gas prices in decimal format to determine the transaction fee (e.g. 0.1uatom)
      --generate-only            Build an unsigned transaction and write it to STDOUT (when enabled, the local Keybase is not accessible)
  -h, --help                     help for instantiate-contract
      --keyring-backend string   Select keyrings backend (os|file|kwallet|pass|test|memory) (default "os")
      --keyring-dir string       The client Keyring directory; if omitted, the default 'home' directory will be used
      --label string             A human-readable name for this contract in lists
      --ledger                   Use a connected Ledger device
      --no-admin                 You must set this explicitly if you dont want an admin
      --node string              <host>:<port> to tendermint rpc interface for this chain (default "tcp://localhost:26657")
      --note string              Note to add a description to the transaction (previously --memo)
      --offline                  Offline mode (does not allow any online functionality
  -o, --output string            Output format (text|json) (default "json")
      --proposal string          Proposal file path (if this path is given, other proposal flags are ignored)
      --run-as string            The address that pays the init funds. It is the creator of the contract and passed to the contract as sender on proposal execution
  -s, --sequence uint            The sequence number of the signing account (offline mode only)
      --sign-mode string         Choose sign mode (direct|amino-json), this is an advanced feature
      --timeout-height uint      Set a block timeout height to prevent the tx from being committed past a certain height
      --title string             Title of proposal
      --type string              Permission of proposal, types: store-code/instantiate/migrate/update-admin/clear-admin/text/parameter_change/software_upgrade
  -y, --yes                      Skip tx broadcasting prompt confirmation
```

## Contract Migration

Migration is the process through which a given smart contract's code can be swapped out or 'upgraded'.

When instantiating a contract, there is an optional admin field that you can set. If it is left empty,
the contract is immutable. If the admin is set (to an external account or governance contract), that
account can trigger a migration. The admin can also reassign the admin role, or even make the contract
fully immutable if desired. However, keep in mind that when migrating from an old contract to a new
contract, the new contract needs to be aware of how the state was previously encoded.

A more detailed description of the technical aspects of migration can be found in the
[CosmWasm migration documentation](https://docs.cosmwasm.com/docs/smart-contracts/migration).
