# Litecoin node with Fulcrum

Run a Litecoin full node along with Fulcrum, using Docker Compose.
In this FAQ, we explain why you might want this, what the system requirements are, and, most importantly, how to go about it.

## Why would I want this?

In general, hosting your node makes much more sense compared to using a remote one *not* controlled by you.
If anything, you do not have to trust other Litecoin nodes or Electrum servers, and, as a major bonus, you transact more privately.
Also, syncing your wallets against *your* node is significantly faster, especially when the clients and the node are in the same local network -- or even when you are syncing over the clearnet.

## Why go the Docker route?

Installing and updating Docker containers is simple and easy.
Such containers can automatically start on boot and also restart automatically, e.g., in case of a non-major issue.
Finally, Docker containers are quite portable across different Linux distributions (and macOS).

## What about system requirements?

- You need a Linux distribution or macOS.
  This whole setup has been successfully tested first on a Raspberry Pi 4 with Debian Bookworm, and then on a laptop with openSUSE Leap 15.6.
- You need a system with at least 4 GiB of RAM.
  Our test Raspberry Pi had 4 GiB of RAM, and our test laptop had 16 GiB of RAM.
- Regarding the disk type, you need at least an SSD, though an NVMe is much preferable.
  At the time of this writing (November 2024), the Litecoin blockchain is about 207 GiB, while the accompanying Fulcrum data directory consumes about 41 GiB of disk space.
- The [Docker Engine must be installed](https://docs.docker.com/engine/install), along with the [Docker Compose plugin](https://docs.docker.com/compose/install).
  In case you are running openSUSE Leap, use this [wiki page](https://en.opensuse.org/Docker).

## How do I set everything up?

Make sure you are in a disk partition with enough free space;
in this project, we opted for bind mounts instead of named volumes, for this is an easier setup in the common scenario of a Raspberry Pi that boots off of an SD card and has a (relatively) large SSD drive attached to it via USB.

Clone this repository and change into the new local directory:

```bash
git clone https://github.com/colder-is-better/litecoind-fulcrum.git && cd litecoind-fulcrum
```

Configure RPC authentication.
For that, use the `rpcauth.py` [script](https://github.com/litecoin-project/litecoin/blob/master/share/rpcauth/rpcauth.py) from the Litecoin source repository.
Download the script directly and make it executable, like so:

```bash
wget https://raw.githubusercontent.com/litecoin-project/litecoin/refs/heads/master/share/rpcauth/rpcauth.py && chmod +x rpcauth.py
```

Pick a username, for example `litecoinrpc`, and have `rpcauth.py` construct the string you will put into your `litecoin.conf`:

```console
$ ./rpcauth.py litecoinrpc
String to be appended to litecoin.conf:
rpcauth=litecoinrpc:90e3b15c2dfa8042b6434a2203282299$11b72c7fe2a215d2cab26f07469ca06dcc25f55bb892940fe7af08f8e7183209
Your password:
dUJYPGmHFhlMENLYgbblvcXvTEioYQVBS4lnrQhx9MA=
```

Note that, in addition to the string for `litecoin.conf`, `rpcauth.py` generated and returned a strong password.
Keep in mind that running `rpcauth.py` again produces a *different* string for `litecoin.conf`, even if the username remains the same;
that's because each time the script runs, it generates a new password and salt.

Now, copy the existing `litecoin.conf.change_me` file and name it `litecoin.conf`:

```bash
cp litecoin.conf.change_me litecoin.conf
```

Using your favorite text editor, open `litecoin.conf`, delete the last line (`rpcauth=run_rpcauth.py_and_change_me`), and replace it with the string `rpcauth.py` gave you.
In our example, `litecoin.conf` will look like this:

```bash
# Accept JSON-RPC commands
server=1

# Disable regression testing network
regtest=0

# Look up transactions included in the blockchain by hash
txindex=1

# Do not host a wallet
disablewallet=1

# Do not create a log file (debug.log)
nodebuglogfile=1

# Setup RPC authentication
rpcauth=litecoinrpc:90e3b15c2dfa8042b6434a2203282299$11b72c7fe2a215d2cab26f07469ca06dcc25f55bb892940fe7af08f8e7183209
```

Save the changes to `litecoin.conf`, then open the `.env` file for editing.
In there, replace the value of `RPC_USER` with the username you have already selected (e.g., `litecoinrpc`).
Similarly, replace the value of `RPC_PASSWORD` with the password `rpcauth.py` generated (e.g., `dUJYPGmHFhlMENLYgbblvcXvTEioYQVBS4lnrQhx9MA=`).
In our example, `.env` will look like this:

```bash
# litecoind RPC authentication
# ----------------------------

RPC_USER=litecoinrpc
RPC_PASSWORD=dUJYPGmHFhlMENLYgbblvcXvTEioYQVBS4lnrQhx9MA=

# Fulcrum initial sync memory
# ---------------------------

# Read more about the --utxo-cache parameter here:
# https://github.com/cculianu/Fulcrum/blob/master/doc/unix-man-page.md#options
# The UTXO_CACHE variable should be set to the most memory you
# can afford Fulcrum, to drastically improve initial sync times.

UTXO_CACHE=2048
```

You are now ready to start the Litecoin daemon along with Fulcrum.

## How do I start the services?

From the `litecoind-fulcrum` directory, issue the following command:

```bash
docker compose up -d
```

## How do I see what is happening behind the scenes?

You may live-watch the logs of `litecoind` and `Fulcrum`, like so:

```bash
docker compose logs -f
```

Hit `Ctrl+C` to stop watching.
To live-watch the logs of `litecoind` only, type:

```bash
docker compose logs -f litecoind
```

Similarly, to live-watch the logs of `Fulcrum` only, type:

```bash
docker compose logs -f fulcrum
```

## New container images are available -- how do I update?

Switch to the `litecoind-fulcrum` directory and type:

```bash
docker compose pull && docker compose up -d
```

## Should I open any ports?

By default, the `litecoind` P2P port (`9333/TCP`) is open for *every* network interface, and the `litecoind` RPC port (`9332/TCP`) is open *only* for the loopback interface.
`fulcrum` communicates with `litecoind` via RPC, and exposes ports `60001/TCP` (non-SSL connections) and `60002/TCP` (SSL connections) for clients to use.

If you are running this setup at home and you want your Litecoin node to be discoverable by other *nodes* on the network, then on your residential router you should forward port `9333/TCP`.
Finally, in case you want to connect a non-local client (e.g., a mobile wallet of yours) to Fulcrum, then on your router you should forward port `60002/TCP`.

## Credits

This repository was mostly based on the work of [sethforprivacy](https://github.com/sethforprivacy) regarding [easy-fulcrum](https://github.com/sethforprivacy/easy-fulcrum) and [docker-litecoind](https://github.com/sethforprivacy/docker-litecoind). And of course, none of that would be possible without [cculianu](https://github.com/cculianu)'s [Fulcrum](https://github.com/cculianu/Fulcrum).
