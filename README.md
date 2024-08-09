# Writing-a-DEFi-script


To create a DeFi script that builds upon an initial token swap and incorporates at least one additional DeFi protocol, let's use the following procedure:

Token Swap: We'll use Uniswap to swap tokens on the Ethereum Sepolia testnet.
Lending Protocol: We'll interact with the Aave protocol to deposit the swapped tokens as collateral.
Prerequisites
Node.js and npm should be installed.
An Ethereum wallet with Sepolia testnet ETH is required.
Set up an Infura project to connect to the Sepolia testnet.

Install necessary packages:
bash
Copy code
npm install ethers @uniswap/sdk @aave/protocol-js dotenv
Step 1: Set Up Environment Variables
Create a .env file in your project directory to securely store your private key and Infura project ID:

        plaintext
Copy code
PRIVATE_KEY=your-private-key
INFURA_PROJECT_ID=your-infura-project-id
Step 2: Writing the Script
Create a file named defi_script.js:

        javascript
Copy code
require('dotenv').config();
const { ethers } = require('ethers');
const { Token, Fetcher, Route, Trade, TokenAmount, TradeType, Percent } = require('@uniswap/sdk');
const { LendingPool, utils: AaveUtils } = require('@aave/protocol-js');

const provider = new ethers.providers.InfuraProvider('sepolia', process.env.INFURA_PROJECT_ID);
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

async function main() {
    // Define token addresses (use appropriate Sepolia testnet token addresses)
    const tokenA = new Token(1, '0x...TokenA_Address', 18, 'TOKENA', 'Token A');
    const tokenB = new Token(1, '0x...TokenB_Address', 18, 'TOKENB', 'Token B');

    // Fetch pair data from Uniswap
    const pair = await Fetcher.fetchPairData(tokenA, tokenB, provider);
    const route = new Route([pair], tokenA);

    // Set up the trade on Uniswap
    const trade = new Trade(
        route,
        new TokenAmount(tokenA, ethers.utils.parseUnits('1', 18)), // Swap 1 TOKENA
        TradeType.EXACT_INPUT
    );

    const slippageTolerance = new Percent('50', '10000'); // 0.5% slippage
    const amountOutMin = trade.minimumAmountOut(slippageTolerance).raw;
    const path = [tokenA.address, tokenB.address];
    const to = wallet.address;
    const deadline = Math.floor(Date.now() / 1000) + 60 * 20; // 20 minutes from now

    // Swap tokens using Uniswap
    const uniswap = new ethers.Contract(
        '0xUniswapRouterAddress',
        ['function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external returns (uint256[] memory amounts)'],
        wallet
    );

    const swapTx = await uniswap.swapExactTokensForTokens(
        ethers.utils.parseUnits('1', 18), // Swap 1 TOKENA
        amountOutMin,
        path,
        to,
        deadline,
        { gasLimit: ethers.utils.parseUnits('200000', 'wei') }
    );

    console.log('Swap Transaction hash:', swapTx.hash);

    // Wait for swap transaction to be mined
    const swapReceipt = await swapTx.wait();
    console.log('Swap transaction mined in block:', swapReceipt.blockNumber);

    // Now deposit the swapped tokens into Aave
    const aaveLendingPoolAddress = '0x...AaveLendingPoolAddress';
    const aaveLendingPool = new ethers.Contract(
        aaveLendingPoolAddress,
        ['function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) public'],
        wallet
    );

    const amountToDeposit = ethers.utils.parseUnits('0.9', 18); // Deposit 0.9 TOKENB to Aave

    const depositTx = await aaveLendingPool.deposit(
        tokenB.address,
        amountToDeposit,
        wallet.address,
        0, // referral code
        { gasLimit: ethers.utils.parseUnits('200000', 'wei') }
    );

    console.log('Aave Deposit Transaction hash:', depositTx.hash);

    // Wait for the deposit transaction to be mined
    const depositReceipt = await depositTx.wait();
    console.log('Deposit transaction mined in block:', depositReceipt.blockNumber);
}

main().catch(console.error);
Explanation of the Script
Token Swap on Uniswap:

We define the token addresses and use Uniswap's SDK to set up a trade.
The script swaps a specified amount of TOKENA for TOKENB using the Uniswap protocol.
Deposit to Aave:

After swapping the tokens, the script deposits the swapped TOKENB into Aave as collateral.
This involves interacting with Aave’s lending pool contract.
Step 3: Running the Script
Make sure your wallet has Sepolia ETH.
Run the script using Node.js:
bash

node defi_script.js

+------------------------+
|        Start           |
+------------------------+
            |
            
            v
            
+------------------------+
|  Fetch Pair Data       |
|   from Uniswap         |
+------------------------+
            |
            
            v
            
+------------------------+
|   Create Trade         |
|   (Token A -> Token B) |
+------------------------+
            |
            
            v
            
+------------------------+
|  Swap Tokens on        |
|   Uniswap              |
+------------------------+
            |
            
            v
            
+------------------------+
| Check Transaction      |
| Receipt                |
+------------------------+
            |
            
            v
            
+------------------------+
| Deposit Token B        |
|  to Aave               |
+------------------------+
            |
            
            v
            
+------------------------+
| Check Deposit Receipt  |
+------------------------+
            |
            
            v
            
+------------------------+
|         End            |
+------------------------+
