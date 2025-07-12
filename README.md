
# Quickstart: Cross-chain USDC Transfer

## Overview

This example uses [web3.js](https://web3js.readthedocs.io/en/v1.8.1/getting-started.html) to transfer USDC from an address on ETH testnet to another address on AVAX testnet.

### Prerequisite
The script requires [Node.js](https://nodejs.org/en/download/) installed.

## Usage
1. Install dependencies by running `npm install`
2. Create a `.env` file and add below variables to it:
    ```js
    ETH_TESTNET_RPC=<ETH_TESTNET_RPC_URL>
    AVAX_TESTNET_RPC=<AVAX_TESTNET_RPC_URL>
    ETH_PRIVATE_KEY=<ADD_ORIGINATING_ADDRESS_PRIVATE_KEY>
    AVAX_PRIVATE_KEY=<ADD_RECEIPIENT_ADDRESS_PRIVATE_KEY>
    RECIPIENT_ADDRESS=<ADD_RECEIPIENT_ADDRESS_FOR_AVAX>
    AMOUNT=<ADD_AMOUNT_TO_BE_TRANSFERED>
    ```
3. Run the script by running `npm run start`

## Script Details
The script has 5 steps:
1. First step approves `ETH_TOKEN_MESSENGER_CONTRACT_ADDRESS` to withdraw USDC from our eth address.
    ```js
    const approveTx = await usdcEthContract.methods.approve(ETH_TOKEN_MESSENGER_CONTRACT_ADDRESS, amount).send({gas: approveTxGas})
    ```

2. Second step executes `depositForBurn` function on `TokenMessengerContract` deployed in [Goerli testnet](https://goerli.etherscan.io/address/0xd0c3da58f55358142b8d3e06c1c30c5c6114efe8)
    ```js
    const burnTx = await ethTokenMessengerContract.methods.depositForBurn(amount, AVAX_DESTINATION_DOMAIN, destinationAddressInBytes32, USDC_ETH_CONTRACT_ADDRESS).send();
    ``` 

3. Third step extracts `messageBytes` emitted by `MessageSent` event from `depositForBurn` transaction logs and hashes the retrieved `messageBytes` using `keccak256` hashing algorithm
    ```js
    const transactionReceipt = await web3.eth.getTransactionReceipt(burnTx.transactionHash);
    const eventTopic = web3.utils.keccak256('MessageSent(bytes)')
    const log = transactionReceipt.logs.find((l) => l.topics[0] === eventTopic)
    const messageBytes = web3.eth.abi.decodeParameters(['bytes'], log.data)[0]
    const messageHash = web3.utils.keccak256(messageBytes);
    ```

4. Fourth step polls the attestation service to acquire signature using the `messageHash` from previous step.
    ```js
    let attestationResponse = {status: 'pending'};
    while(attestationResponse.status != 'complete') {
        const response = await fetch(`https://iris-api-sandbox.circle.com/attestations/${messageHash}`);
        attestationResponse = await response.json()
        await new Promise(r => setTimeout(r, 2000));
    }
    ```

5. Last step calls `receiveMessage` function on `TokenMessengerContract` deployed in [Avalanche Fuji Network](https://testnet.snowtrace.io/address/0xa9fb1b3009dcb79e2fe346c16a604b8fa8ae0a79) to receive USDC at AVAX address.

    *Note: The attestation service is rate-limited, please limit your requests to less than 1 per second.*
    ```js
    const receiveTx = await avaxMessageTransmitterContract.receiveMessage(receivingMessageBytes, signature);


    ```

✅ What's the problem?
Blockchains like Ethereum, Avalanche, and others usually can't talk to each other directly — they work in isolated environments (called “silos”). That means assets and data can't naturally move between them.

🤝 Cosmos is an exception
In the Cosmos ecosystem, chains are designed from the ground up to talk to each other using a protocol called IBC (Inter-Blockchain Communication). So Cosmos-based chains can natively send tokens or messages to one another.

❌ But Ethereum and Avalanche can't do that
Ethereum and Avalanche don’t share a native way to communicate directly. So if you want to send tokens like USDC from Ethereum to Avalanche, you need some kind of bridge.

🌉 What are traditional bridges?
Traditional blockchain bridges were created to solve this problem. They allow you to move assets like USDC from one chain to another, usually in one of two ways:

Lock-and-Mint

You lock the USDC on chain A (e.g., Ethereum).

A new, “wrapped” version of that USDC is minted on chain B (e.g., Avalanche).

Example: If you lock 100 USDC on Ethereum, you get 100 “Wrapped USDC” on Avalanche.

Liquidity Pool Bridging

Instead of minting, bridges use liquidity pools on both chains.

You send USDC to the bridge on Ethereum, and it gives you USDC from its pool on Avalanche.

Think of it like a bank with vaults in two cities — you deposit in one, and they give you cash from the other.

⚠️ What's the issue with these bridges?
They require USDC to be locked up in third-party smart contracts, which reduces how efficiently that money can be used elsewhere.

They introduce trust risks:

You must trust the bridge not to get hacked.

You must trust the bridge operators not to cheat.

These bridges are also targets for attacks, and many have been exploited in the past.

🎯 Why CCTP is better
Circle’s Cross-Chain Transfer Protocol (CCTP) improves on this by burning USDC on the source chain and minting real, native USDC on the destination chain — removing the need for wrapping or pooled liquidity.

Let me know if you’d like me to explain how CCTP works in a simple flow too!
