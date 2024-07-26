Operation Lifecycle

The following document outlines the operation lifecycle. The purpose of this document is to guide you through the process of an operation, from when it is initiated by a user to when it’s fully completed and finalised.

At the bottom you will find a flowchart illustrating the various states an operation goes through. On the left hand side of the diagram you can see the possible states numbered and we will refer to these throughout.
Hub & Spoke Architecture
We use the “Hub & Spoke” model. There will be a series of “Spokes” that connect outlying Blockchains to a central “Hub” Blockchain. Below is a diagram illustrating the architecture.


 
The Hub Chain and Spoke Chains communicate back and forth using a Generic Messaging Protocol (GMP). In our case the GMPs we use are Wormhole and Chainlink CCIP. We use Circle CCTP to facilitate the Cross-Chain native USDC transfers.
Operations
We define an operation to be an action a user can take in the protocol e.g. inviting an address, depositing, borrowing, etc. 

There are three categories of operations:
Operations initiated from SpokeCommon where no “value” is being transferred.
Operations initiated from SpokeToken where “value” is being transferred.
Operations initiated from Hub. 
See When We Don't Care about Finality for an explanation as to what “value” means and its implications. 

For the purposes of this document, we will only consider the first two categories. These are operations which are initiated from a Spoke Chain so must utilise a GMP to communicate with the Hub Chain. The third category’s operations are initiated on the Hub Chain and are thus straightforward because no cross-chain communication is needed.

Operations can also be split into one’s which require a round trip of messages and one’s which don’t. 
An operation is initiated from the Spoke Chain.
A message is sent from the Spoke Chain to the Hub Chain.
A message is received on the Hub Chain.
A message is sent from the Hub Chain back to the Spoke Chain.
A message is received on the Spoke Chain.
Operations which require a round trip have the last two additional steps.
SpokeCommon Operations
The operations initiated from SpokeCommon where no “value” is being transferred.
CreateAccount
InviteAddress
AcceptInviteAddress
UnregisterAddress
AddDelegate
RemoveDelegate
CreateLoan
DeleteLoan
Withdraw
Borrow
RepayWithCollateral
SwitchBorrowType

State 3 will be immediate because no finality is needed on the spoke chain.

The Withdraw and Borrow operations have return messages so progress from state 6 to state 7. All these other operations listed go from state 6 to state 11 because they don’t require a round trip.
SpokeToken Operations
The operations initiated from SpokeCommon where “value” is being transferred:
CreateLoanAndDeposit
Deposit
Repay

State 3 will be delayed because finality is needed on the spoke chain.

In normal circumstances these operations go from state 6 to state 11 because they don’t require a round trip. However if there was a caught reversion at state 6, and the user cancels the operation, then the operation will go from state 6 to state 7. If the user retries the operation and it’s successful, then the operation will go from state 6 to state 11. 
Failed States
There are two failed states, in state 5 and state 9. These should not occur. If they do then something went wrong in the validation of the cross-chain messaging such as someone trying to fabricate a message.

Sometimes the message will run out of gas before the validation can complete. This occurs in state 4 and 8 when an insufficient gas limit was used to relay the message. Depending on the GMP used:
The Wormhole relayer on the message source chain can be called to relay the message with a higher gas limit.
The CCIP relayer on the message destination chain can be called to relay the message with a higher gas limit.

Sometimes the message will be validated but it failed to be handled by the receiver. This occurs in state 6 and 8. See Errors & Recovery for more detail. The operation can be retried or even cancelled depending on the operation. 
























