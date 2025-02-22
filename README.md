# Description

Analogue of the [tonweb js](https://github.com/toncenter/tonweb) library. Feel free to use their docs and examples.

# Notes

- tonsdk/provider part is dirty.

# Installation

```code
$ pip install tonsdk
```

# Usage examples

### Create mnemonic, init wallet class, create external message to deploy the wallet

```python
from tonsdk.contract.wallet import WalletVersionEnum, Wallets
from tonsdk.utils import bytes_to_b64str
from tonsdk.crypto import mnemonic_new


wallet_workchain = 0
wallet_version = WalletVersionEnum.v3r2
wallet_mnemonics = mnemonic_new()

_mnemonics, _pub_k, _priv_k, wallet = Wallets.from_mnemonics(
    wallet_mnemonics, wallet_version, wallet_workchain)
query = wallet.create_init_external_message()
base64_boc = bytes_to_b64str(query["message"].to_boc(False))

print("""
Mnemonic: {}

Raw address: {}

Bounceable, url safe, user friendly address: {}

Base64boc to deploy the wallet: {}
""".format(wallet_mnemonics,
           wallet.address.to_string(),
           wallet.address.to_string(True, True, True),
           base64_boc))
```

### Clients usage example (dirty)

*Note - to use these clients you should install tvm_valuetypes and aiohttp packages*

```python
from abc import ABC, abstractmethod
import asyncio
import aiohttp

from tonsdk.provider import ToncenterClient, SyncTonlibClient, prepare_address, address_state
from tonsdk.utils import TonCurrencyEnum, from_nano


class AbstractTonClient(ABC):
    @abstractmethod
    def _run(self, to_run, *, single_query=True):
        raise NotImplemented

    def get_address_information(self, address: str,
                                currency_to_show: TonCurrencyEnum = TonCurrencyEnum.ton):
        return self.get_addresses_information([address], currency_to_show)[0]

    def get_addresses_information(self, addresses,
                                  currency_to_show: TonCurrencyEnum = TonCurrencyEnum.ton):
        if not addresses:
            return []

        tasks = []
        for address in addresses:
            address = prepare_address(address)
            tasks.append(self.provider.raw_get_account_state(address))

        results = self._run(tasks, single_query=False)

        for result in results:
            result["state"] = address_state(result)
            if "balance" in result:
                if int(result["balance"]) < 0:
                    result["balance"] = 0
                else:
                    result["balance"] = from_nano(
                        int(result["balance"]), currency_to_show)

        return results


class TonCenterTonClient(AbstractTonClient):
    def __init__(self):
        self.loop = asyncio.get_event_loop()
        self.provider = ToncenterClient(base_url="https://testnet.toncenter.com/api/v2/",
                                        api_key="eb542b65e88d2da318fb7c163b9245e4edccb2eb8ba11cabda092cdb6fbc3395")

    def _run(self, to_run, *, single_query=True):
        try:
            return self.loop.run_until_complete(
                self.__execute(to_run, single_query))

        except Exception:  # ToncenterWrongResult, asyncio.exceptions.TimeoutError, aiohttp.client_exceptions.ClientConnectorError
            raise

    async def __execute(self, to_run, single_query):
        timeout = aiohttp.ClientTimeout(total=5)

        async with aiohttp.ClientSession(timeout=timeout) as session:
            if single_query:
                to_run = [to_run]

            tasks = []
            for task in to_run:
                tasks.append(task["func"](
                    session, *task["args"], **task["kwargs"]))

            return await asyncio.gather(*tasks)


class TonLibJsonTonClient(AbstractTonClient):
    def __init__(self):
        self.loop = asyncio.get_event_loop()
        self.provider = SyncTonlibClient(config="./.tonlibjson/testnet.json",
                                         keystore="./.tonlibjson/keystore",
                                         cdll_path="./.tonlibjson/linux_libtonlibjson.so")  # or macos_libtonlibjson.dylib
        self.provider.init()

    def _run(self, to_read, *, single_query=True):
        try:
            if not single_query:
                queries_order = {query_id: i for i,
                                 query_id in enumerate(to_read)}
                return self.provider.read_results(queries_order)

            else:
                return self.provider.read_result(to_read)

        except Exception:  # TonLibWrongResult, TimeoutError
            raise


addr_info = TonCenterTonClient().get_address_information(
    "EQAhE3sLxHZpsyZ_HecMuwzvXHKLjYx4kEUehhOy2JmCcHCT")
print(addr_info)

```
