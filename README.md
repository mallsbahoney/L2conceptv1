# L2conceptv1
BY OPENING/VIEWING: YOU AGREE TO NOT SHARE OR DUPLICATE.
ALL RIGHTS RESERVED.



Users make a PDA once. 

PDA includes these details:
Token addresses of the tokens held by PDA
Amount of each token being held 

This creates them a PDA with token balances. 

The PDA is delegated to MagicBlock. 

Program allows users to increase other token balances in other PDA’s by decreasing the balance in theirs. 

A balance cannot be increased without the account triggering it to be increased has their account decreased.

If a user wants to withdraw the held tokens from their balance, they simply must update the PDA’s state first before withdrawing.

This ensures that if a withdrawal is ever made, the latest state of the PDA is guaranteed to be used to check balances. 

This ensures no one can double or over spend & allows users to change balances without gas (just a one time set up).

CPI is possible with any other account delegated to Magic block and can use this program as a sort of Magic block wallet.

So:
-Every balance PDA is delegated
	•	All transfers are internal ledger updates inside ER
	•	Withdraw requires:
	•	Commit
	•	L1 SPL transfer
	•	Balance re-check

Which means double-spend protection works because:
	•	PDA state is single-source-of-truth
	•	Withdrawal always references committed state
	•	Internal debits must precede credits

Contract wise it’s pretty straightforward:

First use:
Connect wallet
Press “Join”

Set up:
Program owns all PDAs.
User makes presses “Join” to
make PDA. 
User is given text fields to input token addresses (the main SOL token addresses are always included as default) + optional balances (they all start at 0). 
Once they’re done, they press “Complete”

Pressing Complete creates the PDA With the token addresss details and saves the wallet address of the wallet making the PDA. 

Anytime a user wants to increase their own token balance: they can do so by sending the update through L1 and sending the same amount of tokens along with it to the program (so that the program can hold it).

A state balance update from magic block is required with every withdrawal request (so that it’s guaranteed to always use updated balances). 

Once a user has their PDA wallet, web app shows the balance via standard wallet like interface.

In the interface, when a user presses send, they can include as many addresses as they want (and even use a feature called “batch send” to upload a mass amount of wallet names in a big text box and simply separate them by comma)

The wallet will let them know anytime they send to an address that’s not been delegated to MagicBlock (as that will have to be on L1) and if a user still wants to send: then they have to do a state update first and then the txn is sent from the program balance as normal. 

If they send to any other account that IS delegated: it’s near instant and only requires signature now (no gas, only gas required during state updates). 

1) “Program holds tokens”

On Solana, the program cannot “hold tokens” literally. What I mean is: 
	•	A vault ATA per mint (or escrow account per mint) owned by a vault authority PDA.
	•	Deposits/withdrawals are SPL Token CPIs to/from that vault ATA.

So the deposit instruction is: transfer user_ata -> vault_ata then update ledger.

2) “User inputs token addresses”

Letting users input arbitrary mints creates operational problems:
	•	Storage growth / resizing PDAs
	•	Batch transfers require passing recipient PDAs + possibly mint metadata; you’ll hit transaction limits
	•	You risk listing scam mints / weird token programs / frozen mints unless you validate

It should feel like simply putting in addresses in the UX, but on the back end it should be not a literal list at all—store balances in per-mint subaccounts: user_balance_pda(owner, mint).

3) “Withdrawal requires MagicBlock state update first”

We can’t enforce “user clicked state update in UI.” You can only enforce an on-chain condition. So to achieve this, we do:

Option A (recommended): Withdraw only allowed when not delegated
	•	While delegated to ER, the account is effectively “in session” and you disallow withdraw.
	•	Withdraw requires undelegate/commit, then withdraw on L1.

Minimal instruction set that matches your doc

Accounts
	•	UserState { owner, state_version, ... } (PDA per user)
	•	UserBalance { owner, mint, amount, version } (PDA per user per mint) recommended
	•	VaultAuthority (PDA)
	•	VaultATA(mint) (ATA owned by VaultAuthority)

Instructions
	•	join() → create UserState
	•	add_mint(mint) → create UserBalance(owner, mint) (if you want opt-in)
	•	deposit(mint, amount) L1 → SPL transfer into vault, then user_balance += amount
	•	transfer_batch(mint, [(to_user_state, to_user_balance, amount)...]) → atomic debit/credit (ER if all delegated)
	•	commit/undelegate → finalize ER state back to L1
	•	withdraw(mint, amount, destination_ata) L1 only → require committed/undelegated, then SPL transfer out of vault

Batch send is possible, but bounded

Yes, you can do “mass update many balances in one instruction,” but you will hit:
	•	account count limits per transaction
	•	transaction size limits
	•	compute limits

So “upload 10,000 recipients” must be automatically chunked into many txs.

Implementation pattern:
	•	client splits into batches of N recipients (e.g., 10–25 depending on account footprint)
