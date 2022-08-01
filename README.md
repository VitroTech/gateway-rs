The Helium Gateway application is a service designed to run on Linux-based LoRaWAN gateways.

It's intended to run alongside a typical LoRa packet forwarder and to connect via Semtech's Gateway Messaging Protocol (GWMP, using JSON v1 or v2).

In turn, the Helium Gateway application does two things:

- fetches blockchain context, such as routing tables and OUI endpoints, from a `Gateway Service`; this means the application does not need to maintain a full ledger of copy of the blockchain
- connects and routes packets to the appropriates OUI endpoints (ie: `Helium Routers`)

```
                                                                 +-----------+
+-----------+                       +------------+               |  Gateway  |
|           |                       |            |<--- gRPC ---->|  Service  |
|  packet   |<--- Semtech GWMP ---->|   Helium   |               +-----------+
| forwarder |       over UDP        |   Gateway  |               +-----------+
|           |                       |            |<--- gRPC ---->|  Helium   |
+-----------+                       +------------+               |  Routers  |
                                                                 +-----------+
```

The current gateway project forwards packets to the router via state channels and is eligible for data rewards **only**. Proof of coverage is not yet possible.

## Installing

   The default region of the gateway is `US915`, if your region is different you can set the right one in `/etc/helium_gateway/settings.toml`. Just add the following line at the beginning of the file:

   ```shell
   region = "<region>"
   ```

   Possible values are : `US915| EU868 | EU433 | CN470 | CN779 | AU915 | AS923_1 | AS923_2 | AS923_3 | AS923_4 | KR920 | IN865`. After updating the value you need to restart the application.

If this command succeeds the logs on the gateway will show the service starting and the local packet forwarder client connecting to the gateway service.

## Building

For building the application for Crystal these are the following commands:

1. Install `rust`
   ```shell
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```
2. Install cargo `cross`, `cargo-make`, and, if needed, `cargo-deb`. The `cross` command allows for cross compiling to hardware targets using docker images, while the `cargo-make` command is used to package up. If creating a deb package, `cargo-deb` is also needed.
   ```shell
   cargo install cross
   cargo install cargo-make
   cargo install cargo-deb
   ```
3. Build the application or package using one of the following:

   1. Build the application binary using the target triplet from the supported targets. Note the use of the `--release` flag to optimize the target binary for size. Debug builds may be to large to run on some targets.

      ```shell
      cross build --target armv7-unknown-linux-gnueabihf --release
      ```

      The resulting application binary is located in

      ```
      target/armv7-unknown-linux-gnueabihf/release/helium_gateway
      ```

   2. Copy the binary to Crystal using sftp or scp

   3. Run the program on Crystal with 

   ```
   ./helium_gateway server
   ```


## Additional usage info

The Helium Gateway application can be configured to suit your hardware/software environment in a variety of ways - either from the command line, using customizations to the `settings.toml` file or with environment variables. The following sections describe this functionality in more detail as well as more general information on how to use the application.

### Settings file

The Helium Gateway application is configured using TOML files for your settings. The defaults can be found in the [default.toml](https://github.com/helium/gateway-rs/blob/main/config/default.toml) file in this repo. You should not edit this file directly, rather you should create a `settings.toml` file and store it either at the default expected locaton `/etc/helium_gateway/settings.toml` or at a custom location of your choosing by passing the `-c` flag to the `helium_gateway` application as shown below in the [general usage section](#general-usage-info).

You can customize any of the fields shown in the [default.toml](https://github.com/helium/gateway-rs/blob/main/config/default.toml) file, however it is important to make sure that when editing you maintain the same ordering as shown in the default file. You do not need to include all fields in the `settings.toml` file - only the ones you want to change from default values - however maintaining the correct sections is highly recommended to avoid any unexpected behaviour.

For example, this config **will not work correctly** as it all ends up in the `update` section:

```
[update]
platform = "ramips_24kec"

[log]
method = "stdio"
level = "info"
timestamp = false

region = "EU868"
```

Whereas this one will:

```
region = "EU868"

[log]
method = "stdio"
level = "info"
timestamp = false

[update]
platform = "ramips_24kec"
```

By maintaining the same layout as the `default.toml` file you can avoid any unexpected errors.

The following are the settings that can be changed and their default and optional values:

```
# can be any file location where you store the gateway_key.bin file
keypair = "/etc/helium_gateway/gateway_key.bin"
# can be any ip address and port combination
listen = "127.0.0.1:1680"
# possible values are : US915| EU868 | EU433 | CN470 | CN779 | AU915 | AS923_1 | AS923_2 | AS923_3 | AS923_4 | KR920 | IN865
region = "US915"

[log]
# either stdio or syslog
method = "stdio"
# possible values are: debug | info | warn
level = "info"
# either true or false
timestamp = false

[update]
# either true or false
enabled = true
# this setting must be overriden to get updates. choose from the supported platforms listed here https://github.com/helium/gateway-rs#supported-platforms
platform = "unknown"
# update channel to use: alpha | beta | release | semver - more details can be found at https://github.com/helium/gateway-rs#releases
channel = "semver"
# Interval in minutes between update checks
interval = 10
# The github release stream to check for updates
uri = "https://api.github.com/repos/helium/gateway-rs/releases"
# The command to run to install the update.
command = "/etc/helium_gateway/install_update"

[cache]
# The location of the cache store for the great gateway service
store = "/etc/helium_gateway/cache"
```

The default gateways / router `uri` and `pubkey` parameters can be changed, but this is only if you are using non-Helium routers. For general use with Helium you should leave these the same.

### Using the ECC crypto chip

If your gateway is enabled with an ECC608 crypto chip which is set up correctly, you can configure helium_gateway to use the crypto chip for secure key storage and crypto operations.

To use in your `settings.toml` override the `keypair` setting to reflect the use of the ECC and specify the bus address and slot to use. For example:

```
keypair = "ecc://i2c-1:96?slot=0&network=mainnet"
```

will have helium_gateway use the ECC at the `/dev/i2c-1` device driver location, use bus address `96` (which is hex `0x60`) and slot `0` for it's crypto operations. While marking the resulting key as a mainnet key. Bus address, slot and network are all optional parameters and default to the above values (only device driver location is required such as `ecc://i2c-1`).

Note that the file based keypair will no longer be used once the ECC is configured for use.

See the [gateway-mfr-rs repo](https://github.com/helium/gateway-mfr-rs) for instructions on configuring, locking, and testing an ECC chip.

### Envrionment variables

Instead of overriding paramaters in the [default.toml](https://github.com/helium/gateway-rs/blob/main/config/default.toml) file using a `settings.toml` file as described above, you can instead use environment variables. The environment variable name will be the same name as the entries in the settings file in uppercase and prefixed with "GW\_". For example, following on from the above example where we change the region using `region = "EU868"` in the settings file, setting an environment variable of `GW_REGION="EU868"` will override the region setting. If the settings are in one of the lower sections such as the `[update]` or `[log]` sections then you need to also include that in the environment variable name such as `GW_LOG_LEVEL` or `GW_UPDATE_PLATFORM`.

The settings are loaded first from `default.toml`, then from a `settings.toml` file, and then from environment variables and any duplicates are overriden in the order. Therefore, please note that if you have a setting in all three locations, the environment variable will override the settings in the other two locations.

### General usage info

Using the Helium Gateway application is pretty simple, and the vast majority of the information you will need to use it can be gleaned by using the `--help` flag which provides the following output:

```
Helium Light Gateway

USAGE:
    helium_gateway [FLAGS] [OPTIONS] <SUBCOMMAND>

FLAGS:
        --daemon     Daemonize the application
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -c <config>        Configuration folder to use. default.toml will be loaded first and any custom settings in
                       settings.toml merged in [default: /etc/helium_gateway]

SUBCOMMANDS:
    add       Construct an add gateway transaction for this gateway
    help      Prints this message or the help of the given subcommand(s)
    key       Commands on gateway keys
    server    Run the gateway service
    update    Commands for gateway updates
```

As you can see, apart from the `help` command, there are four core subcommands that you can pass: `add`, `key`, `server` and `update`. The descriptions of what these subcommands do is shown in brief in the above help output, and are explained in more detail in the sections below.

The only option available is the `config` option using the `-c` flag. This tells the application where your configuration file is located and can be used as follows whilst passing any of the other commands such as `server` or `add` (default is `/etc/helium_gateway`):

```
./helium_gateway -c /location/of/config/folder server
```

Lastly you can check the version, read the help information or daemonize the application using the `--version`, `--help` and `--daemon` flags respectively.

### Add gateway subcommand

As shown in the help output below, this subcommand is used to construct an add gateway transaction which can subsequently be used with the Helium Wallet application to onboard the gateway to the blockchain. More infomation on this process can be found [on the docs article for Data Only Hotspots](https://docs.helium.com/mine-hnt/data-only-hotspots/#add-hotspot).

```
Construct an add gateway transaction for this gateway

USAGE:
    helium_gateway add [OPTIONS] --owner <owner> --payer <payer>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --mode <mode>      The staking mode for adding the light gateway [default: dataonly]
        --owner <owner>    The target owner account of this gateway
        --payer <payer>    The account that will pay account for this addition
```

So for example, to construct an add gateway transaction you would enter the following command at the terminal:

```
./helium_gateway add --owner WALLET_ADDRESS --payer WALLET_ADDRESS
```

You need to substitute WALLET_ADDRESS for the wallet address you will use for the owner of the hotspot and the payer of the transaction fees respectively...but please note that the `--payer` address must be the same as the one you will use to submit the transaction to the blockchain, or the transaction will fail.

The output of this command is a JSON object which looks like the following:

```json
{
  "address": "11TL62V8NYvSTXmV5CZCjaucskvNR1Fdar1Pg4Hzmzk5tk2JBac",
  "fee": 65000,
  "mode": "dataonly",
  "owner": "14GWyFj9FjLHzoN3aX7Tq7PL6fEg4dfWPY8CrK8b9S5ZrcKDz6S",
  "payer": "14GWyFj9FjLHzoN3aX7Tq7PL6fEg4dfWPY8CrK8b9S5ZrcKDz6S",
  "staking fee": 1000000,
  "txn": "CrkBCiEBrlImpYLbJ0z0hw5b4g9isRyPrgbXs9X+RrJ4pJJc9MkSIQA7yIy7F+9oPYCTmDz+v782GMJ4AC+jM+VfjvUgAHflWSJGMEQCIGfugfLkXv23vJcfwPYjLlMyzYhKp+Rg8B2YKwnsDHaUAiASkdxUO4fdS33D7vyid8Tulizo9SLEL1lduyvda9YVRCohAa5SJqWC2ydM9IcOW+IPYrEcj64G17PV/kayeKSSXPTJOMCEPUDo+wM="
}
```

You can also pass a `--mode` flag followed by the hotspot type (`dataonly | light | full`) as shown below:

```
./helium_gateway add --owner WALLET_ADDRESS --payer WALLET_ADDRESS --mode light
```

The output of this command will be mostly the same as if you used the default `dataonly` however you will see that the mode has changed to `"mode": "light",` and the staking fee amount has changed to `"staking fee": 4000000`.

The txn field from the JSON object needs to be used as the input to the wallet command `helium-wallet hotspot add` when you subsequently want to add it to the blockchain. For example, using the above JSON object as an example, you would use the following command:

```
helium-wallet hotspots add CrkBCiEBrlImpYLbJ0z0hw5b4g9isRyPrgbXs9X+RrJ4pJJc9MkSIQA7yIy7F+9oPYCTmDz+v782GMJ4AC+jM+VfjvUgAHflWSJGMEQCIGfugfLkXv23vJcfwPYjLlMyzYhKp+Rg8B2YKwnsDHaUAiASkdxUO4fdS33D7vyid8Tulizo9SLEL1lduyvda9YVRCohAa5SJqWC2ydM9IcOW+IPYrEcj64G17PV/kayeKSSXPTJOMCEPUDo+wM=
```

### Gateway keys subcommand

This subcommand can be used to get the address and animal name of the gateway from the keys file as shown in the help output below. Note that the helium_gateway server has to be running for this command to work.

```
Commands on gateway keys

USAGE:
    helium_gateway key <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    help    Prints this message or the help of the given subcommand(s)
    info    Commands on gateway keys
```

Using this is as simple as passing the following command in a terminal from wherever you installed the `helium_gateway` application:

```
./helium_gateway key info
```

The output of this is a JSON object containing the address and animal name of the hotspot as shown below:

```json
{
  "address": "11TL62V8NYvSTXmV5CZCjaucskvNR1Fdar1Pg4Hzmzk5tk2JBac",
  "name": "wide-neon-kestrel"
}
```

### Gateway server

The gateway server subcommand is used to start the gateway service on your device.

```
Run the gateway service

USAGE:
    helium_gateway server

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
```

Running it is as simple as:

```
./helium_gateway server
```

However as discussed above you can also pass the `-c` option to tell the service that you are using a different location for your config files:

```
./helium_gateway -c /location/of/config/folder server
```

### Gateway update

The gateway update subcommand pretty much does what it says on the tin - it is used to update the software version of the gateway. You can see the help output for this command shown below.

```
Commands for gateway updates

USAGE:
    helium_gateway update <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    download    Download an updates. This does not install the update
    help        Prints this message or the help of the given subcommand(s)
    list        List available updates
```

As you can see from the help output, there are essentially two functions of the `update` command - to list available updates and to download an update.

For basic usage of the `list` function you can simply use:

```
./helium_gateway update list
```

And this will give you an output similar to the following:

```
1.0.0-alpha.10
1.0.0-alpha.13 (*)
1.0.0-alpha.12
1.0.0-alpha.11
1.0.0-alpha.9
1.0.0-alpha.8
```

This takes the default update channel and platform from your environment variables, `settings.toml` or `default.toml` depending on whether you have set any overrides or not. The list will default to a total of 10 update versions, unless you pass a flag to tell it to output a different amount. However, if you want to be more specific you can use something like the following:

```
./helium_gateway update list --channel <channel> --platform <platform> -n <count>
```

Where:

- `<channel>` is the channel to list updates for (alpha | beta | release | semver - more details can be found [here](https://github.com/helium/gateway-rs#releases) - defaults to 'update.channel' setting)
- `<count>` is the number of update entries to list (defaults to 10)
- `<platform>` is the platform to list entries for (choose from the supported platforms listed [here](https://github.com/helium/gateway-rs#supported-platforms) - defaults to 'update.platform' setting)

Lastly, we have the `download` function which can be used as follows:

```
./helium_gateway update download --path <path> <version>
```

Where:

- `<path>` is the directory path to download the update to (defaults to your current directory)
- `<version>` is the version you want to update to (which can be found from the list command described above - there is no default, so the version must be passed or this command will fail)