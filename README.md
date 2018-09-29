<p align="center">
  <img src="./exw3_logo.jpg"/>
</p>

## Installation

```elixir
def deps do
  [{:exw3, "~> 0.2.0"}]
end
```
## Overview

ExW3 is a wrapper around ethereumex to provide a high level, user friendly json rpc api. It currently only supports Http. The primary feature it provides is a handy abstraction for working with smart contracts.

## Usage

Ensure you have an ethereum node to connect to at the specified url in your config. An easy local testnet to use is ganache-cli:
```
ganache-cli
```

Or you can use parity:
Install Parity, then run it with

```
echo > passfile
parity --chain dev --unlock=0x00a329c0648769a73afac7f9381e08fb43dbea72 --reseal-min-period 0 --password passfile
```

If Parity complains about password or missing account, try

```
parity --chain dev --unlock=0x00a329c0648769a73afac7f9381e08fb43dbea72
```

Make sure your config includes:
```elixir
config :ethereumex,
  url: "http://localhost:8545"
```

Currently, ExW3 supports a handful of json rpc commands. Mostly just the useful ones. If it doesn't support a specific commands you can always use the [Ethereumex](https://github.com/exthereum/ethereumex) commands.

Check out the [documentation](https://hexdocs.pm/exw3/ExW3.html)

```elixir
iex(1)> accounts = ExW3.accounts()
["0x00a329c0648769a73afac7f9381e08fb43dbea72"]
iex(2)> ExW3.balance(Enum.at(accounts, 0))
1606938044258990275541962092341162602522200978938292835291376
iex(3)> ExW3.block_number()
1252
iex(4)> simple_storage_abi = ExW3.load_abi("test/examples/build/SimpleStorage.abi")
%{
  "get" => %{
    "constant" => true,
    "inputs" => [],
    "name" => "get",
    "outputs" => [%{"name" => "", "type" => "uint256"}],
    "payable" => false,
    "stateMutability" => "view",
    "type" => "function"
  },
  "set" => %{
    "constant" => false,
    "inputs" => [%{"name" => "_data", "type" => "uint256"}],
    "name" => "set",
    "outputs" => [],
    "payable" => false,
    "stateMutability" => "nonpayable",
    "type" => "function"
  }
}
iex(5)> ExW3.Contract.start_link
{:ok, #PID<0.265.0>}
iex(6)> ExW3.Contract.register(:SimpleStorage, abi: simple_storage_abi)
:ok
iex(7)> {:ok, address, tx_hash} = ExW3.Contract.deploy(:SimpleStorage, bin: ExW3.load_bin("test/examples/build/SimpleStorage.bin"), options: %{gas: 300_000, from: Enum.at(accounts, 0)})
{:ok, "0x22018c2bb98387a39e864cf784e76cb8971889a5",
 "0x4ea539048c01194476004ef69f407a10628bed64e88ee8f8b17b4d030d0e7cb7"}
iex(8)> ExW3.Contract.at(:SimpleStorage, address)
:ok
iex(9)> ExW3.Contract.call(:SimpleStorage, :get)
{:ok, 0}
iex(10)> ExW3.Contract.send(:SimpleStorage, :set, [1], %{from: Enum.at(accounts, 0), gas: 50_000})
{:ok, "0x88838e84a401a1d6162290a1a765507c4a83f5e050658a83992a912f42149ca5"}
iex(11)> ExW3.Contract.call(:SimpleStorage, :get)
{:ok, 1}
```

## Asynchronous

ExW3 now provides async versions of `call` and `send`. They both return a `Task` that can be awaited on.

```elixir
  t = ExW3.Contract.call_async(:SimpleStorage, :get)
  {:ok, data} = Task.await(t)
```

## Events

ExW3 allows the retrieval of event logs using filters. In this example, assume we have already deployed and registered a contract called :EventTester.

```elixir
# We can optionally specify extra parameters like `:fromBlock`, and `:toBlock`
{:ok, filter_id} = ExW3.Contract.filter(:EventTester, "Simple", %{fromBlock: 42, toBlock: "latest"})

# After some point that we think there are some new changes
{:ok, changes} = ExW3.Contract.get_filter_changes(filter_id)

# We can then uninstall the filter after we are done using it
ExW3.Contract.uninstall_filter(filter_id)
```

## Indexed Events

Ethereum allows a user to add topics to filters. This means the filter will only return events with the specific index parameters. For all of the extra options see [here](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_newfilter)

If you have written your event in Solidity like this:
```
    event SimpleIndex(uint256 indexed num, bytes32 indexed data, uint256 otherNum);
```

You can add filter on which logs will be returned back to the RPC client, based on the indexed fields.

ExW3 allows for 2 ways of specifying these parameters or `topics` in two ways. The first, and probably more preferred way, is with a map:

```elixir
  indexed_filter_id = ExW3.Contract.filter(
    :EventTester,
    "SimpleIndex",
    %{
      topics: %{num: 46, data: "Hello, World!"},
    }
  )
```

The other option is with a list, but this is order dependent, and any values you don't want to specify must be represented with a `nil`.

```elixir
  indexed_filter_id = ExW3.Contract.filter(
    :EventTester,
    self(),
    %{
      topics: [nil, "Hello, World!"]
    }
  )
```

In this case we are skipping the `num` topic, and only filtering on the `data` parameter.


# Compiling Solidity

To compile the test solidity contracts after making a change run this command:
```
solc --abi --bin --overwrite -o test/examples/build test/examples/contracts/*.sol
```