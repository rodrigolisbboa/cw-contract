# Deployment Guide

## 1. Compile CosmWasm Contracts

Clone smart contracts starter from github repository

```sh
git clone https://github.com/CosmWasm/cw-plus
```

Go into folder

```sh
cd cw-plus
```

Compile smart contracts inside this directory.

This command is for non ARM(Non-Applie silicon) devices, for example: Windows, Linux

```sh
docker run --rm -v "/User/injective/cw-plus":/code \
--mount type=volume,source="c_cache",target=/code/target \
--mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
cosmwasm/workspace-optimizer:0.12.12
```

This is for Apple silicon devices (M1, M2, etc.) please use:

```sh
docker run --rm -v "/User/injective/cw-plus":/code \
--mount type=volume,source="c_cache",target=/code/target \
--mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
cosmwasm/workspace-optimizer-arm64:0.12.12
```

Note: in these commands, `/User/injective/cw-plus` means absolute path where your code of smart contacts were placed.(it would be current path in case you are following the instructions correctly). So if sample path is not exactly same as yours, pls replace it. `c_cache` can be a string whatever you want. However you can change only the string at first of \_ . so it could be abc_cache, contracts_cache and etc. But I recommend making it meaningful.

This command will compile all smart contracts inside this directory and generate wasm files in new folder, `artifacts`.

## 2. Download Dockerised Injective Chain Binary

If you're on Apple silicon make sure to use an ARM image for injective-core.

```sh
    public.ecr.aws/l9h3g6c6/injective-core:v1.10.1
--> public.ecr.aws/l9h3g6c6/injective-core:v1.10.1-arm
```

Executing this command will make the docker container execute indefinitely.

```sh
docker run --name="injective-core-v1.10.1" \
-v=/User/injective/my-first-contracts/artifacts:/var/artifacts \
--entrypoint=sh public.ecr.aws/l9h3g6c6/injective-core:v1.10.1 \
-c "tail -F anything"-
```

Note: Pls make sure about your absolute path of artifacts folder in this command. In case of wrong path, pls write your own path follow above note.

Open a new terminal and access the Docker container to initialize the chain.

```sh
docker exec -it injective-core-v1.10.1 sh
```

Let’s start by adding the jq dependency, a lightweight command line JSON processor, which we will need later on.

```sh
apk add jq
```

We can now add a user (when prompted use 12345678 as password). We will use it only to generate a new private key that will later on be used to sign messages on the testnet.

```sh
injectived keys add firstuser
```

Save the output. It's information of local account and the phrase of wallet. You will use this accout to deploy smart contracts on mainnet so that you will propably fund this account. (Might be 50 INJ? because this deployment is a proposal which will be submitted to chain and chain will vote this proposal with this minimum deposit amount.)

## 3. Upload the wasm contract

Upload the wasm contract inside of artifacts folder

```sh
injectived tx gov submit-proposal wasm-store /var/artifacts/my_first_contract.wasm \
--title "Title of proposal - Upload contract" \
--description "Andrea's deploy" \
--instantiate-everybody true \
--deposit=1000000000000000000inj \
--run-as inj1myp2jnfel5s0jlsztz275qtjmdfz9efjqsduwe \
--from firstuser \
--gas=10000000 \
--chain-id=injective-1 \
--broadcast-mode=sync \
--yes \
--gas-prices=500000000inj
```

If you need to input password when prompted, you should enter the password of your local account. This will be 12345678 if you follow the instructions correctly.

Note: `/var/artifacts/my_first_contract.wasm` is the absolute path of wasm file that you want to upload to mainnet. So you should change it if it doesn't match with yours. `inj1myp2jnfel5s0jlsztz275qtjmdfz9efjqsduwe` is the address of your local account. `firstuser` is the localaccount name in which you will sign this tx.

Note: Fee amount will be 0.005 INJ. And the minimum deposit amount(might be 50 INJ) is required to have on the wallet is deposit amount to the chain(something like stake, it's not exactly confirmed by the support/dev team, coz they sucks to support the community currently.)
