<div align="center">
  <h1>Relayer</h1>

![banner](./docs/images/comp.gif)

[![Project Status: Initial Release](https://img.shields.io/badge/repo%20status-active-green.svg?style=flat-square)](https://www.repostatus.org/#active)
![GitHub Workflow Status](https://github.com/cosmos/relayer/actions/workflows/build.yml/badge.svg)
[![GoDoc](https://img.shields.io/badge/godoc-reference-blue?style=flat-square&logo=go)](https://godoc.org/github.com/cosmos/relayer)
[![Go Report Card](https://goreportcard.com/badge/github.com/cosmos/relayer)](https://goreportcard.com/report/github.com/cosmos/relayer)
[![License: Apache-2.0](https://img.shields.io/github/license/cosmos/relayer.svg?style=flat-square)](https://github.com/cosmos/relayer/blob/main/LICENSE)
[![Lines Of Code](https://img.shields.io/tokei/lines/github/cosmos/relayer?style=flat-square)](https://github.com/cosmos/relayer)
[![Version](https://img.shields.io/github/tag/cosmos/relayer.svg?style=flat-square)](https://github.com/cosmos/relayer/latest)
</div>

In IBC, blockchains do not directly pass messages to each other over the network. This is where `relayer` comes in. 
A relayer process monitors for updates on opens paths between sets of [IBC](https://ibcprotocol.org/) enabled chains.
The relayer submits these updates in the form of specific message types to the counterparty chain. Clients are then used to 
track and verify the consensus state.

In addition to relaying packets, this relayer can open paths across chains, thus creating clients, connections and channels.

Additional information on how IBC works can be found [here](https://ibc.cosmos.network/).

<div align="center">

   | Relayer |        IBC-Go        |
   |:-------:|:--------------------:|
   | v1.0.0  | ibc-go v1, ibc-go v2 |
   | v2.0.0  | ibc-go v3            |

</div>


**If you are updating the relayer from any version prior to `v2.0.0-rc1`, your current config file is not compatible. See: [config_migration](docs/config_migration.md)

---

## Table Of Contents
- [Basic Usage - Relaying Across Chains](#Basic-Usage---Relaying-Packets-Across-Chains)
- [Create Path Across Chains](./docs/create-path-across-chain.md)
- [Troubleshooting](./docs/troubleshooting.md)
- [Features](./docs/features.md)
- [Relayer Terminology](./docs/terminology.md)
- [Recommended Pruning Settings](./docs/node_pruning.md)
- [Demo](./docs/demo.md)

---
## Basic Usage - Relaying Packets Across Chains

> The `-h` (help) flag tailing any `rly` command will be your best friend. USE THIS IN YOUR RELAYING JOURNEY.

---

1. **Clone, checkout and install the latest release ([releases page](https://github.com/cosmos/relayer/releases)).**

   *[Go](https://go.dev/doc/install) needs to be installed and a proper Go environment needs to be configured*

    ```shell
    $ git clone https://github.com/cosmos/relayer.git
    $ cd relayer && git checkout v2.0.0-rc3
    $ make install
    ```

2. **Initialize the relayer's configuration directory/file.**
 
   ```shell
   $ rly config init --memo "Zuka#5870"
   ``` 
   
   The `--memo` flag is also available for other `rly` commands also that involve sending transactions such as `rly tx link` and `rly start`. It can be passed there to override the `config.yaml` value if desired.

   To omit the memo entirely, including the default value of `rly(VERSION)`, use `-` for the memo.

3. **Configure the chains you want to relay between.**
   
   To add the chain config files manually, example config files have been included [here](https://github.com/cosmos/relayer/tree/main/docs/example-configs/)
   ```
     rly chains add --url https://raw.githubusercontent.com/zukaman/relayer-v2.0.0-rc3/main/configs/stride.json stride
     rly chains add --url https://raw.githubusercontent.com/zukaman/relayer-v2.0.0-rc3/main/configs/GAIA.json GAIA
   ```
   
4. **Import OR create new keys for the relayer to use when signing and relaying transactions.**

   >`key-name` is an identifier of your choosing.    

   If you need to generate a new private key you can use the `add` subcommand.

    ```shell
    $ rly keys add stride [key-name]  
    $ rly keys add GAIA [key-name]  
    ```
  
   If you already have a private key and want to restore it from your mnemonic you can use the `restore` subcommand.

   ```shell
   $ rly keys restore stride [key-name] "mnemonic words here"
   $ rly keys restore GAIA [key-name] "mnemonic words here"
   ```

5. **Edit the relayer's `key` values in the config file to match the `key-name`'s chosen above.**

   >This step is necessary if you chose a `key-name` other than "default"
   
   Example:
      ```yaml
      - type: cosmos
         value:
         key: YOUR-KEY-NAME-HERE
         chain-id: cosmoshub-4
         rpc-addr: http://localhost:26657
      ```

6. **Ensure the keys associated with the configured chains are funded.**

   >Your configured addresses will need to contain some of the respective native tokens for paying relayer fees.  
   
   <br>
   You can query the balance of each configured key by running:  

   ```shell
   $ rly q balance stride
   $ rly q balance GAIA
   ```

7. **Configure path meta-data in config file.**


8. #### **Configure the channel filter.**
   
   By default, the relayer will relay packets over all channels on a given connection.  
   <br>
   Each path has a `src-channel-filter` which you can utilize to specify which channels you would like to relay on.   
   <br>
   The `rule` can be one of three values:  
   - `allowlist` which tells the relayer to relay on _ONLY_ the channels in `channel-list`
   - `denylist` which tells the relayer to relay on all channels _BESIDES_ the channels in `channel-list`
   - empty value, which is the default setting, and tells the relayer to relay on all channels    
   <br>
   
   Since we are only worried about the canonical channel between the Cosmos Hub and Osmosis our filter settings would look like the following.  
   <br>
   Example:
   ```yaml
   hubosmo:
      src:
          chain-id: cosmoshub-4
          client-id: 07-tendermint-259
          connection-id: connection-257
      dst:
          chain-id: osmosis-1
          client-id: 07-tendermint-1
          connection-id: connection-1
      src-channel-filter:
              rule: allowlist
              channel-list: [channel-141]  
   ```
   
   >Because two channels between chains are tightly coupled, there is no need to specify the dst channels.
   >If you only know the "dst" channel-ID you can query the "src" channel-ID by running: `rly q channel <dst_chain_name> <dst_channel_id> <port> | jq '.channel.counterparty.channel_id'`

10. **Finally, we start the relayer on the desired path.**

     The relayer will periodically update the clients and listen for IBC messages to relay.

     ```shell
     $ rly paths list
     $ rly start [path]
     ```
   
    You will need to start a separate shell instance for each path you wish to relay over.

    >When running multiple instances of `rly start`, you will need to use the `--debug-addr` flag and provide an address:port. You can also pass an empty string `''`  to turn off this feature or pass `localhost:0` to randomly select a port.

    ---
    [[TROUBLESHOOTING](docs/troubleshooting.md)]
---

## Security Notice

If you would like to report a security critical bug related to the relayer repo,
please reach out @jackzampolin or @Ethereal0ne on telegram.

## Code of Conduct

The Cosmos community is dedicated to providing an inclusive and harassment free
experience for contributors. Please visit [Code of Conduct](CODE_OF_CONDUCT.md) for more information.

---

[Create Path Across Chains -->](docs/create-path-across-chain.md)
