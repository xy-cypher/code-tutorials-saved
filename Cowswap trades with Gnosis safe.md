Source: https://hackmd.io/@2jvugD4TTLaxyG3oLkPg-g/H14TQ1Omt
# [Cowswap](https://cowswap.exchange/) trades with Gnosis safe

In this document we are going to discuss how to execute Cowswap transactions using the new presign functionality. This feature allows smart contracts to use cowswap.

To batch actions in a single transaction I am going to use [ape-safe](https://github.com/banteg/ape-safe). At Yearn, ape-safe is used extensively to test transactions before sending them to the multisign to approve.

## Approving the tokens to sell

The first step is approving the tokens you want to sell to the *GPv2VaultRelayer*.

The safe that I am using had yvUSDC, yvYFI and yvLINK, so my first tx takes care of withdrawing and approving each token. Code is the following:

`import click`

`from ape_safe import ApeSafe`

`from brownie *`

`def withdraw_and_approve_tokens_sep_21():`

    # Get the safe
    safe = ApeSafe("0xMySafeAddress")

    # Contract we need to approve so our tokens can be transferFrom
    gnosis_vault_relayer = safe.contract("0xC92E8bdf79f0507f65a392b0ab4667716BFE0110")

    # yearn vault tokens
    yvUSDC = safe.contract("0x5f18C75AbDAe578b483E5F43f12a39cF75b973a9")
    yvYFI = safe.contract("0xE14d13d8B3b85aF791b2AADD661cDBd5E6097Db1")
    yvLINK = safe.contract("0x671a912C10bba0CFA74Cfc2d6Fba9BA1ed9530B2")

    for vault in [yvUSDC, yvYFI, yvLINK]:
        print(f"Processing {vault.name()}")

        # Withdraw everything from the vault
        vault.withdraw()
        token = safe.contract(vault.token())
        token_balance = token.balanceOf(safe.address)
        print(f"Balance of {token.name()}: {(token_balance / 10 ** token.decimals()):_}")
        assert token.balanceOf(safe.address) > 0

        # Approve so we can create a cowswap order
        token.approve(gnosis_vault_relayer, 2**256-1)


    safe_tx = safe.multisend_from_receipts()
    account = click.prompt("signer", type=click.Choice(accounts.load()))
    safe_tx.sign(accounts.load(account).private_key)
    safe.preview(safe_tx, events=False, call_trace=False)
    safe.post_transaction(safe_tx)
    
After testing in a fork, ape-safe will ask for your account password and submit the tx to the multisign.

After the tx is executed, we would have our vanilla erc-20 tokens plus all approvals needed to submit an order.

## Creating the order

Let’s do an intermediate step and create a method to submit the order. The gist of the process is the following:

- Get the quote of the trade to do
- Create an order through the api and get an order id
- Use the order id to set a flag on-chain, saying you are ok with that trade

I tried adding comments around the code

`from time import time`

`import requests`

`import web3`

`def cowswap_sell(safe, sell_token, buy_token):`

    # Contract used to sign the order
    gnosis_settlement = safe.contract("0x9008D19f58AAbD9eD0D60971565AA8510560ab41")
    amount = sell_token.balanceOf(safe.address)

    # get the fee + the buy amount after fee
    fee_and_quote = "https://protocol-mainnet.gnosis.io/api/v1/feeAndQuote/sell"
    get_params = {
        "sellToken": sell_token.address,
        "buyToken": buy_token.address,
        "sellAmountBeforeFee": amount
    }
    r = requests.get(fee_and_quote, params=get_params)
    assert r.ok and r.status_code == 200

    # These two values are needed to create an order
    fee_amount = int(r.json()['fee']['amount'])
    buy_amount_after_fee = int(r.json()['buyAmountAfterFee'])
    assert fee_amount > 0
    assert buy_amount_after_fee > 0

    # Pretty random order deadline :shrug:
    deadline = chain.time() + 60*60*24*100 # 100 days

    # Submit order
    order_payload = {
        "sellToken": sell_token.address,
        "buyToken": buy_token.address,
        "sellAmount": str(amount-fee_amount), # amount that we have minus the fee we have to pay
        "buyAmount": str(buy_amount_after_fee), # buy amount fetched from the previous call
        "validTo": deadline,
        "appData": web3.Web3.keccak(text="yearn goes moooooo").hex(), # required field, do not change :)
        "feeAmount": str(fee_amount),
        "kind": "sell",
        "partiallyFillable": False,
        "receiver": safe.address,
        "signature": safe.address,
        "from": safe.address,
        "sellTokenBalance": "erc20",
        "buyTokenBalance": "erc20",
        "signingScheme": "presign" # Very important. this tells the api you are going to sign on chain
    }
    orders_url = f"https://protocol-mainnet.gnosis.io/api/v1/orders"
    r = requests.post(orders_url, json=order_payload)
    assert r.ok and r.status_code == 201
    order_uid = r.json()
    print(f"Payload: {order_payload}")
    print(f"Order uid: {order_uid}")

    # With the order id, we set the flag, basically signing as the gnosis safe.
    gnosis_settlement.setPreSignature(order_uid, True)
    
## Running the tx from the safe

The last step is running a tx submission to the multisign using the previous method.

`def run_trade_sep_21():`

    safe = ApeSafe("0xMySafeAddress")

    yfi = safe.contract("0x0bc529c00C6401aEF6D220BE8C6Ea1667F6Ad93e")
    usdc = safe.contract("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48")
    link = safe.contract("0x514910771AF9Ca656af840dff83E8264EcF986CA")
    gusd = safe.contract("0x056Fd409E1d7A124BD7017459dFEa2F387b6d5Cd")

    cowswap_sell(safe, yfi, gusd)
    cowswap_sell(safe, usdc, gusd)
    cowswap_sell(safe, link, gusd)

    safe_tx = safe.multisend_from_receipts()
    account = click.prompt("signer", type=click.Choice(accounts.load()))
    safe_tx.sign(accounts.load(account).private_key)
    safe.preview(safe_tx, events=False, call_trace=False)
    safe.post_transaction(safe_tx)
    
This run will not be the typical ape safe tx. When you run the code you are actually calling the api and creating an order. You can create as many orders as you want BUT they are not valid until you sign them. In our case, executing the tx in the multisign will do the trick.

Once the tx is signed, you can query the `/solvable_orders` endpoint to see your trades avaiable to solvers. You can check the output from https://protocol-rinkeby.dev.gnosisdev.com/api/#/default/get_api_v1_solvable_orders.

After a few minutes I got two transfers, the first was `yfi` and `link` converted to `gusd` and on the second one `usdc` converted to `gusd`.

## Conclusion

Cowswap is a great way of doing trades. The more users go through Cowswap the more cows we will see.

Now with the smart contract integration, there is no excuse not to use Cowswap from Gnosis, or even from your smart contract.

_This code and explanation wouldn’t have been possible without the help from Felix and Nicholas <3_
