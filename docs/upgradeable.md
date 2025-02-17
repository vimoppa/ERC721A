# Using with Upgrades

If you are deploying upgradeable contracts, 
such as using [OpenZeppelin Upgrade Plugins](https://docs.openzeppelin.com/upgrades-plugins/1.x/), 
you will need to use the upgradeable variant of ERC721A. 

For more information, please refer to 
[OpenZeppelin's documentation](https://docs.openzeppelin.com/contracts/4.x/upgradeable).

Since v4, the upgradeable variant uses the Diamond storage pattern as defined in [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535).

## Installation

```
npm install --save-dev erc721a-upgradeable
```

## Usage

The package shares the same directory layout as the main ERC721A package, but every file and contract has the suffix `Upgradeable`.

Constructors are replaced by internal initializer functions following the naming convention `__{ContractName}__init`. 

These functions are internal, and you must define your own public initializer function that calls the parent class' initializer.

```solidity
pragma solidity ^0.8.4;

import 'erc721a-upgradeable/contracts/ERC721AUpgradeable.sol';

contract Something is ERC721AUpgradeable {
    function initialize() initializerERC721A public {
        __ERC721A_init('Something', 'SMTH');
    }

    function mint(uint256 quantity) external payable {
        // _safeMint's second argument now takes in a quantity, not a tokenId.
        _safeMint(msg.sender, quantity);
    }
}
```

Note the modifier `initializerERC721A` on the `initialize()` function.

If using with another upgradeable library, please do use their respective initializer modifier on the `initialize()` function, in addition to the `initializerERC721A` modifier.

## Deployment

If you are using hardhat, you can deploy it using 
[OpenZeppelin Upgrade Plugins](https://docs.openzeppelin.com/upgrades-plugins/1.x/).

```
npm install --save-dev @openzeppelin/hardhat-upgrades
```

**Deploy Script**

```javascript
// scripts/deploy.js
const { ethers, upgrades } = require('hardhat');
const fs = require('fs');

async function main () {
    const Something = await ethers.getContractFactory('Something');
    console.log('Deploying...');
    const something = await upgrades.deployProxy(
        Something, 
        [], 
        { initializer: 'initialize' }
    );
    await something.deployed();
    const addresses = {
        proxy: something.address,
        admin: await upgrades.erc1967.getAdminAddress(something.address), 
        implementation: await upgrades.erc1967.getImplementationAddress(
            something.address)
    };
    console.log('Addresses:', addresses);

    try { 
        await run('verify', { address: addresses.implementation });
    } catch (e) {}

    fs.writeFileSync('deployment-addresses.json', JSON.stringify(addresses));
}

main();
```

**Upgrade Script**

```javascript
// scripts/upgrade.js
const { ethers, upgrades } = require('hardhat');
const fs = require('fs');

async function main () {
    const Something = await ethers.getContractFactory('Something');
    console.log('Upgrading...');
    let addresses = JSON.parse(fs.readFileSync('deployment-addresses.json'));
    await upgrades.upgradeProxy(addresses.proxy, Something);
    console.log('Upgraded');

    addresses = {
        proxy: addresses.proxy,
        admin: await upgrades.erc1967.getAdminAddress(addresses.proxy), 
        implementation: await upgrades.erc1967.getImplementationAddress(
            addresses.proxy)
    };
    console.log('Addresses:', addresses);
    
    try { 
        await run('verify', { address: addresses.implementation });
    } catch (e) {}

    fs.writeFileSync('deployment-addresses.json', JSON.stringify(addresses));
}

main();
```

### Local

Add the following to your `hardhat.config.js`:

```javascript
// hardhat.config.js
require("@nomiclabs/hardhat-waffle");
require('@openzeppelin/hardhat-upgrades');

module.exports = {
    solidity: "0.8.11"
};
```

**Deploy**

```
npx hardhat run --network localhost scripts/deploy.js
```

**Upgrade**

```
npx hardhat run --network localhost scripts/upgrade.js
```

### Testnet / Mainnet

We will use the Rinkeby testnet as an example.

Install the following packages if they are not already installed:

```
npm install --save-dev @nomiclabs/hardhat-etherscan
npm install --save-dev dotenv
```

Add the following to your environment file `.env`:

```
ETHERSCAN_KEY="Your Etherscan API Key"
PRIVATE_KEY="Your Wallet Private Key"
RPC_URL_RINKEBY="https://Infura Or Alchemy URL With API Key"
```

Add the following to your `hardhat.config.js`:

```javascript
// hardhat.config.js
require("@nomiclabs/hardhat-waffle");
require('dotenv').config();
require('@openzeppelin/hardhat-upgrades');
require("@nomiclabs/hardhat-etherscan");

module.exports = {
	solidity: "0.8.11",
	networks: {
		rinkeby: {
			url: process.env.RPC_URL_RINKEBY,
			accounts: [process.env.PRIVATE_KEY],
			gasPrice: 100000000000,
			gasLimit: 1000000000,
		}
	},
	etherscan: {
		// Your API key for Etherscan
		// Obtain one at https://etherscan.io/
		apiKey: process.env.ETHERSCAN_KEY,
	}
};
```

**Deploy**

```
npx hardhat run --network rinkeby scripts/deploy.js
```

**Upgrade**

```
npx hardhat run --network rinkeby scripts/upgrade.js
```
