## Reusable Github workflow for solana programs

This repository provides GitHub workfloes that automatically build, verify, and deploy solana programs including their IDL uploads.
There is also squads multisig support which is highly recommended to use.
Here you can find a [detailed guide](https://github.com/solana-developers/squads-program-action) on how to use the squads integration. For the workflow you just need to set `use-squads` to true and add the needed secrets.

### Features

- ✅ Automated program builds
- ✅ Program verification against source code
- ✅ IDL buffer creation and uploads
- ✅ Squads multisig integration
- ✅ Program deploys for both devnet and mainnet
- ✅ Compute budget optimization
- ✅ Retry mechanisms for RPC failures

### How to use

```yaml
name: Devnet Build and Deploy

on:
  workflow_dispatch:
    inputs:
      priority_fee:
        description: "Priority fee for transactions"
        required: true
        default: "300000"
        type: string

jobs:
  build:
    uses: solana-developers/github-workflows/.github/workflows/reusable-build.yaml@main
    with:
      program: "hello_world"
      network: "devnet"
      deploy: true
      upload_idl: false
      verify: false
      use-squads: false
      priority-fee: ${{ github.event.inputs.priority_fee }}
    secrets:
      DEVNET_SOLANA_DEPLOY_URL: ${{ secrets.DEVNET_SOLANA_DEPLOY_URL }}
      DEVNET_DEPLOYER_KEYPAIR: ${{ secrets.DEVNET_DEPLOYER_KEYPAIR }}
      PROGRAM_ADDRESS_KEYPAIR: ${{ secrets.PROGRAM_ADDRESS_KEYPAIR }}
```

Or for mainnet with verification:

```yaml
name: Release to mainnet with IDL and verify

on:
  workflow_dispatch:
    inputs:
      priority_fee:
        description: "Priority fee for transactions"
        required: true
        default: "300000"
        type: string

jobs:
  build:
    uses: solana-developers/github-workflows/.github/workflows/reusable-build.yaml@main
    with:
      program: "transaction_example"
      network: "mainnet"
      deploy: true
      upload_idl: true
      verify: true
      use-squads: false
      priority-fee: ${{ github.event.inputs.priority_fee }}

    secrets:
      MAINNET_SOLANA_DEPLOY_URL: ${{ secrets.MAINNET_SOLANA_DEPLOY_URL }}
      MAINNET_DEPLOYER_KEYPAIR: ${{ secrets.MAINNET_DEPLOYER_KEYPAIR }}
      PROGRAM_ADDRESS_KEYPAIR: ${{ secrets.PROGRAM_ADDRESS_KEYPAIR }}
      MAINNET_MULTISIG: ${{ secrets.MAINNET_MULTISIG }}
      MAINNET_MULTISIG_VAULT: ${{ secrets.MAINNET_MULTISIG_VAULT }}
```

There are two examples:

- [Anchor Program](https://github.com/Woody4618/anchor-github-action-example)
- [Native Program](https://github.com/Woody4618/native-solana-github-action-example)

### Required Secrets for specific actions

Some of the options of the build workflow require you to add secrets to your repository:

```bash
# Network RPC URLs
DEVNET_SOLANA_DEPLOY_URL=   # Your devnet RPC URL - Recommended to use a payed RPC url
MAINNET_SOLANA_DEPLOY_URL=  # Your mainnet RPC URL - Recommended to use a payed RPC url

# Deployment Keys
DEVNET_DEPLOYER_KEYPAIR=    # Base58 encoded keypair for devnet
MAINNET_DEPLOYER_KEYPAIR=   # Base58 encoded keypair for mainnet

PROGRAM_ADDRESS_KEYPAIR=    # Keypair of the program address - Needed for initial deploy and for native programs to find the program address. Can also be overwritten in the workflow if you dont have the keypair.

# For Squads integration (There is sadly no devnet squads ui)
MAINNET_MULTISIG=          # Mainnet Squads multisig address
MAINNET_MULTISIG_VAULT=    # Mainnet Squads vault address
```

### Extend and automate

You can easily extend or change your workflow. For example run the build workflow automatically on every push to a development branch.

```bash
  push:
    branches:
      - develop
      - dev
      - development
    paths:
      - 'programs/**'
      - 'Anchor.toml'
      - 'Cargo.toml'
      - 'Cargo.lock'
```

Or run a new release to mainnet on every tag push for example.

```bash
  push:
    tags:
      - 'v*'
```

Or you can setup a matrix build for multiple programs and networks.
Customize the workflow to your needs!

### Running the actions locally

If you for some reason want to run the actions locally you can do so with the following commands using the act command.

Follow the instructions [here](https://nektosact.com/installation/index.html) to install act.

1. Build

You need to copy the workflow file to your local `.github/workflows` directory because act does not support reusable workflows.
Just pick the parameters you want. This is using act to run the workflow locally. Good for testing or if you dont want to install anything because this is running in docker and outputs the build artifacts as well.

```bash
act -W .github/workflows/reusable-build.yaml \
 --container-architecture linux/amd64 \
 --secret-file .secrets \
 workflow_dispatch \
 --input program=transaction-example \
 --input network=devnet \
 --input deploy=true \
 --input upload_idl=true \
 --input verify=true \
 --input use-squads=false
```

2. Run anchor tests

Note: The anchor tests use solana-test-validator which does not work in act docker container on mac because of AVX dependency. Wither run them in github, locally without docker or open PR to fix it. I couldnt find a nice way to fix it.
You can adjust the workflow to run your specific tests as well.

```bash
act -W .github/workflows/test.yaml \
 --container-architecture linux/amd64 \
 --secret-file .secrets \
 workflow_dispatch \
 --input program=transaction-example
```

## How to setup Squads integration:

In general its recommended to use the [Squads Multisig](https://docs.squads.so/squads-cli/overview) to manage your programs.
It makes your program deployments more secure and is considered good practice.

1. Setup a new squad in [Squads](https://v4.squads.so/squads/) then transfer your program authority to the squad.

2. Add your local keypair to the squad as a member (At least needs to be a voter) so that you can propose transactions. And also add that keypair as a github secret.
   To run it locally add the following to your .secrets file:

![alt text](image.png)

```bash
DEVNET_DEPLOYER_KEYPAIR=
MAINNET_DEPLOYER_KEYPAIR=
```

2. Add the following to your .secrets file if you want to run it locally or add them to your github secrets if you want to run it in github actions:

```bash
DEVNET_MULTISIG=
DEVNET_MULTISIG_VAULT=
MAINNET_MULTISIG=
MAINNET_MULTISIG_VAULT=
```

Where Multisig vault is the address you can find on the top left corner in the [Squads Dachboard](https://v4.squads.so/squads/)
The MULTISIG is the address of the multisig you want to use this one you can find the the settings. Its a bit more hidden so that people dont accidentally use it as program upgrade authority.

What this will do is write a program and an IDL buffer for your program and then propose a transaction that you can approve in the Squads UI.

4. Now you can run the workflow with the following command:

```bash
act -W .github/workflows/build.yaml \
 --container-architecture linux/amd64 \
 --secret-file .secrets \
 workflow_dispatch \
 --input program=transaction-example \
 --input network=devnet \
 --input deploy=true \
 --input upload_idl=true --input use-squads=true --input verify=true
```

## 📝 Todo List

### Program Verification

- [x] Trigger verified build PDA upload
- [x] Verify build remote trigger
- [x] Support and test squads Verify
- [x] Support and test squads IDL
- [x] Support and test squads Program deploy

### Action Improvements

- [x] Separate IDL and Program buffer action
- [x] Remove deprecated cache functions
- [x] Remove node-version from anchor build
- [x] Skip anchor build when native program build
- [ ] Make verify build and anchor build in parallel
- [x] Trigger release build on tag push
- [x] Trigger devnet releases on develop branch?
- [x] Make solana verify also work locally using cat
- [x] Use keypairs to find deployer address to remove 2 secrets
- [x] Add priority fees
- [x] Add extend program if needed
- [x] Bundle the needed TS scripts with the .github actions for easier copy paste

### Testing & Integration

- [x] Add running tests
  - Research support for different test frameworks
- [ ] Add Codama support
- [ ] Add to solana helpers or mucho -> release

Close Buffer:

You may need this in case your deploy failed and you want to close a buffer that was already transfered to your multisig.

```bash
solana program show --buffers --buffer-authority <You multisig vault address>

npx ts-node scripts/squad-closebuffer.ts \
 --rpc "https://api.mainnet-beta.solana.com" \
 --multisig "FJviNjW3L2u2kR4TPxzUNpfe2ZjrULCRhQwWEu3LGzny" \
 --buffer "7SGJSG8aoZj39NeAkZvbUvsPDMRcUUrhRhPzgzKv7743" \
 --keypair ~/.config/solana/id.json \
 --program "BhV84MZrRnEvtWLdWMRJGJr1GbusxfVMHAwc3pq92g4z"
```
