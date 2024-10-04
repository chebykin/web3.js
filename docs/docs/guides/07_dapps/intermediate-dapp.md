---
sidebar_position: 3
sidebar_label: 'Tutorial: Intermediate dApp Development'
---

# Intermediate dApp Development

This tutorial demonstrates using Web3.js to build a dApp that uses the [EIP-6963 standard](https://eips.ethereum.org/EIPS/eip-6963). EIP-6963 was designed to make it easy for dApp developers to support users with more than one wallet browser extension. Rather than relying on the global `window.ethereum` object, EIP-6963 specifies a mechanism that allows multiple wallet providers to announce their availability to a dApp. The dApp in this tutorial will allow the user to transfer ether from one of their wallet accounts to another account on the network.

:::tip
If you encounter any issues while following this guide or have any questions, don't hesitate to seek assistance. Our friendly community is ready to help you out! Join our [Discord](https://discord.gg/F4NUfaCC) server and head to the **#web3js-general** channel to connect with other developers and get the support you need.
:::

## Step 1: Prerequisites

This tutorial assumes basic familiarity with the command line as well as familiarity with React and [Node.js](https://nodejs.org/). Before starting this tutorial, ensure that Node.js and its package manager, npm, are installed.

```console
$: node -v
# your version may be different, but it's best to use the current stable version
v20.14.0
$: npm -v
10.8.2
```

Make sure that at least one EIP-6963 compliant wallet browser extension is installed and set up, such as:

- [Enkrypt](https://www.enkrypt.com/download.html)
- [Exodus](https://www.exodus.com/download/)
- [MetaMask](https://metamask.io/download/)
- [Trust Wallet](https://trustwallet.com/download)

This tutorial will use MetaMask as an example.

## Step 2: Initialize a New React Project and Add Dependencies

Initialize a new React project and navigate into the new project directory:

```console
npx create-react-app web3-eip6963-dapp --template typescript
cd web3-eip6963-dapp
```

Add Web3.js to the project with the following command:

```console
npm i web3
```

This tutorial uses a local [Hardhat](https://hardhat.org/) network, which will be configured to fund the wallet's account. To support this, install [Hardhat](https://hardhat.org/) and [ts-node](https://typestrong.org/ts-node/) as development dependencies:

```console
npm i -D hardhat ts-node
```

## Step 3: Configure and Start the Hardhat Node

Because Hardhat will be configured with the wallet's secret recovery phrase, it's important that the Hardhat configuration file is not checked into version control systems like GitHub. Open the `.gitignore` file that was created by the `create-react-app` command and add a line for `hardhat.config.js`:

```bash
# See https://help.github.com/articles/ignoring-files/ for more about ignoring files.

hardhat.config.js

# dependencies
/node_modules
/.pnp
.pnp.js

# testing
/coverage

# production
/build

# misc
.DS_Store
.env.local
.env.development.local
.env.test.local
.env.production.local

npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

Create a file called `hardhat.config.js` and add the following Hardhat configuration:

```typescript
module.exports = {
  networks: {
    hardhat: {
      accounts: {
        mnemonic: "<SECRET RECOVER PHRASE>",
      },
    },
  },
};
```

Replace `<SECRET RECOVER PHRASE>` with the wallet's [secret recovery phrase](https://support.metamask.io/privacy-and-security/how-to-reveal-your-secret-recovery-phrase/).

Start the Hardhat development network by executing the following command:

```console
npx hardhat node
```

Executing this command will produce the following output, which provides the URL that can be used to connect to the Hardhat development network:

```console
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/

Accounts
========
Account #0: <ACCOUNT 0 ADDRESS> (10000 ETH)

...

Account #19: <ACCOUNT 19 ADDRESS> (10000 ETH)
```

:::note
If the Hardhat development network was properly configured with the wallet's secret recovery phrase, `<ACCOUNT ADDRESS 0>` should match the address of the wallet's first account.
:::

The Hardhat development network needs to remain running in the terminal that was used to start it. Open a new terminal instance in the project directory to execute the remaining commands in this tutorial.

## Update the React App and Create a Provider Store

Delete the following files:

- `src/App.css`
- `src/App.test.tsx`
- `src/logo.svg`

Create a `src/useProviders.ts` file and add the following code:

```ts
import { useSyncExternalStore } from "react";
import { providers, Web3 } from "web3";

// initial empty list of providers
let providerList: providers.EIP6963ProviderDetail[] = [];

/**
 * External store for subscribing to EIP-6963 providers
 */
const providerStore = {
  // get current list of providers
  getSnapshot: () => providerList,
  // subscribe to EIP-6963 provider events
  subscribe: (callback: () => void) => {
    // update the list of providers
    function setProviders(response: providers.EIP6963ProviderResponse) {
      providerList = [];
      response.forEach((provider: providers.EIP6963ProviderDetail) => {
        providerList.push(provider);
      });

      // notify subscribers that the list of providers has been updated
      callback();
    }

    // Web3.js helper function to request EIP-6963 providers
    Web3.requestEIP6963Providers().then(setProviders);

    // handler for newly discovered providers
    function updateProviders(
      providerEvent: providers.EIP6963ProvidersMapUpdateEvent,
    ) {
      setProviders(providerEvent.detail);
    }

    // register handler for newly discovered providers with Web3.js helper function
    Web3.onNewProviderDiscovered(updateProviders);

    // return a function that unsubscribes from the created event listener
    return () =>
      window.removeEventListener(
        providers.web3ProvidersMapUpdated as any,
        updateProviders,
      );
  },
};

// export the provider store as a React hook
export const useProviders = () =>
  useSyncExternalStore(providerStore.subscribe, providerStore.getSnapshot);
```

This file exports a single member - a React [`useSyncExternalStore` hook](https://react.dev/reference/react/useSyncExternalStore) with a subscription to the EIP-6963 providers. The provider store uses the Web3.js types and helper functions for working with the EIP-6963 standard. Any React component can use this hook to access a dynamic list of the available EIP-6963 providers.

Replace the contents of the `src/App.tsx` file with the following:

```tsx
import type { providers } from "web3";
import { useProviders } from "./useProviders";

function App() {
  // get the dynamic list of providers
  const providers = useProviders();

  return (
    <>
      {providers.map((provider: providers.EIP6963ProviderDetail) => {
        // list available providers
        return (
          <div key={provider.info.uuid}>
            {provider.info.name} [{provider.info.rdns}]
          </div>
        )
      })}
    </>
  );
}

export default App;
```

Use the `npm start` command to launch the dApp in a new browser window. If everything is working properly, all available EIP-6963 providers should be listed.

## Use a Provider with Web3.js

Replace the contents of the `src/App.tsx` file with the following:

```tsx
import { type providers, Web3 } from "web3";
import { useProviders } from "./useProviders";
import { useEffect, useState } from "react";

function App() {
  // get the dynamic list of providers
  const providers = useProviders();

  // application state
  const [web3, setWeb3] = useState<Web3 | undefined>(undefined);
  const [accounts, setAccounts] = useState<string[]>([]);
  const [balances, setBalances] = useState<Map<string, bigint>>(new Map());

  function setProvider(provider: providers.EIP6963ProviderDetail) {
    const web3: Web3 = new Web3(provider.provider);
    setWeb3(web3);
    web3.eth.requestAccounts().then(setAccounts);
    provider.provider.on("chainChanged", () => window.location.reload());
  }

  // update account balances
  useEffect(() => {
    async function updateBalances() {
      if (web3 === undefined) {
        return;
      }

      const balances = new Map();
      for (const account of accounts) {
        const balance = await web3.eth.getBalance(account);
        balances.set(account, balance);
      }

      setBalances(balances);
    }

    updateBalances();

    if (web3 === undefined) {
      return;
    }

    const subscription = web3.eth
      .subscribe("newBlockHeaders")
      .then((subscription) => {
        subscription.on("data", updateBalances);
        return subscription;
      });

    return () => {
      subscription.then((subscription) => subscription.unsubscribe());
    };
  }, [accounts, web3]);

  return (
    <>
      {web3 === undefined
        ? providers.map((provider: providers.EIP6963ProviderDetail) => {
            // list available providers
            return (
              <div key={provider.info.uuid}>
                <button
                  onClick={() => setProvider(provider)}
                  style={{ display: "inline-flex", alignItems: "center" }}
                >
                  <img
                    src={provider.info.icon}
                    alt={provider.info.name}
                    width="35"
                  />
                  <span> {provider.info.name}</span>
                </button>
              </div>
            );
          })
        : accounts.map((address: string) => {
            return (
              <div key={address}>
                {address}: {`${balances.get(address)}`}
              </div>
            );
          })}
    </>
  );
}

export default App;
```
