# Advanced Tutorial (Loan Contract)

The aim of this tutorial is to introduce you to the advanced concepts of Smart Contracts in Vault and how to approach building Smart Contracts which satisfy business requirements.

In this advanced tutorial we are going to show some advanced techniques that can be used to write a Smart Contract. For this purpose, we present a hypothetical scenario where a loan product needs to be created. We are going to assume that the loan product allows customers to borrow money from a bank, and the bank expects to be repaid in equal, monthly instalments for the duration of the contract.

To achieve such behaviour, we are going to show how a Smart Contract can initialise the movement of funds to credit another account specified by the customer. Furthermore we are going to present how to understand and use balances in contracts, and how they can be used to achieve business requirements.

In the next part of the tutorial, we are going to set up some scheduled events, show how these events interact with Vault, and explore the business logic that can be achieved with them.

Finally, we are going to show how to use account notes, which are useful when you want to notify the account holder about some specific event which happened with their Smart Contract.

## The Purpose of this Advanced Smart Contract Tutorial
- Show how to create movements of funds between accounts

- Show how to perform re-balancing of balance addresses based on incoming postings

- Show advanced business logic implementation using Scheduler

- Show the use of Account Notes

- Show the use of Contract Simulation

- Show some tips & tricks commonly used in Smart Contracts

## High Level Requirements For A Personal Loan
For this tutorial, we are going to assume hypothetical business requirements for writing a Smart Contract for a personal loan.

Requirements:
- Customers should be able to borrow between 1,000 and 20,000 GBP
- Customers can specify which account they want to have the funds moved to
- Duration of the loan must be between 1 and 5 years
- The loan should allow a variable interest rate based on the amount borrowed, which will then fall into one of 5 defined tiers as per table shown below:

|Tier |Min Amount|Max Amount|Gross Rate|
|-----|----------|----------|----------|
|tier1|1,000     |2,999	    |13.50%    |
|tier2|3,000     |4,999	    |9.80%     |
|tier3|5,000     |7,499	    |4.50%     |
|tier4|7,500     |14,999    |3.00%     |
|tier5|15,000    |20,000    |3.50%     |

- If a customer has not repaid the expected monthly amount:
  - Apply a fee
  - Send a notification to customer for the missed repayment
- Repayments can be broken down into multiple transactions, but cannot be summed up higher than the total required repayment amount

- Interest on the outstanding loan amount should be accrued at the end of every day, with accrual precision of 4 decimal places

- Charging of the interest should happen once a month at the start of the day of expected repayment, with application precision of 2 decimal places

## Requirement mapping
---

Let us get started with the requirements for a loan product we want to write. We will look at each requirement one by one and consider how they should be represented in the Smart Contract code. The first thing to look for in the requirements is if there are any potential Contract Parameters that have to be created. Once we extract all Contract Parameters from them, the next thing is to consider what actions need to be performed, and at what moment of the account lifecycle. These two steps are repeated for each requirement.

### Customers should be able to borrow between 1,000 and 20,000 GBP

*Parameters*: In this first requirement, we could make use of two parameters to hold the information we need. The first one is the amount to be borrowed (which will be a decimal value with boundaries set to `1,000` and `20,000`). The second one is the currency that we allow the money to be borrowed in, which in this case will be a string value set to `GBP`.

> Although the string `GBP` could just be there in the code without a parameter, this does not follow the best practices and should be put in a parameter.

*Actions*: In terms of actions required, it would involve creating a posting instruction which moves funds between a loan account and some destination account specified by the customer. That action should happen at the beginning of the account lifecycle, which suggests the best place to put it would be in the `post-activate` hook, which is run after the account is opened.

### Customers can specify which account they want to have the funds moved to.

*Parameters*: This requirement is pretty straightforward, we would need to create a Contract Parameter which would store information about the account number the user wants to move money to.

*Actions*: No actions needed. This action would be covered by the action from previous requirement. The Contract Parameter created here would be used to determine which account to send money to.

### Duration of the loan must be between 1 and 5 years

*Parameters*: Again, this requirement is similar to the previous one, we would need to create a Contract Parameter which would store information about the loan duration in years and have a value between `1` and `5`.

*Actions*: No actions needed.

### The loan should allow a variable interest rate based on the amount borrowed, which will then fall into one of 5 defined tiers as per table shown below:

|Tier |Min Amount|Max Amount|Gross Rate|
|-----|----------|----------|----------|
|tier1|1,000     |2,999	    |13.50%    |
|tier2|3,000     |4,999	    |9.80%     |
|tier3|5,000     |7,499	    |4.50%     |
|tier4|7,500     |14,999    |3.00%     |
|tier5|15,000    |20,000    |3.50%     |

*Parameters*: This requirement is similar to previous ones, even though it looks more complicated. We are asked to create some way to store information about interest rates based on amount borrowed by the customer. That means that we should use a Contract Parameter or multiple Contract Parameters to save the data from this table.

*Actions*: No actions needed.

### If a customer has not repaid the expected monthly amount:
- Apply a fee
- Send a notification to customer for the missed repayment

*Parameters*: From the requirement description we can see there is only one Contract Parameter, which would be the fee value.

*Actions*: From an action point of view, this task requires applying a fee to the account when a customer has not paid the necessary monthly amount. That suggests that there should be a check at some point in time, which would check if the expected amount has been paid. So, we would need to add a scheduled event to be run periodically and make this check. If the amount was not repaid, it would apply a fee and send a notification to the customer.

### Repayments can be broken down into multiple transactions, but cannot be summed up higher than the total required repayment amount

Parameters: Does not require Contract Parameters.

Actions: For actions, this requirement states that we cannot take payment above the specified amount and in the contract lifecycle this check should happen when a new posting is attempted, so we can check the total repayments so far and reject the posting if the total will be higher than the required amount. That logic would be in the pre-posting hook.

### Interest on the outstanding loan amount should be accrued at the end of every day, with accrual precision of 4 decimal places

*Parameters*: Does not require Contract Parameters.

*Actions*: To complete this task we will need to create a scheduled event, which will be executed at the end of each day. Then, we need to code the logic for accruing interest, and making funds transfer for that accrual. These actions would happen in two places: firstly, after the account is opened, the scheduled events schedule would be created for the given account. Secondly, we need to add logic to be executed whenever the scheduled event is run. That would correspond to execution_schedules and scheduled_code hooks, respectively.

### Charging of the interest should happen once a month at the start of the day of expected repayment, with application precision of 2 decimal places

*Parameters*: Does not require Contract Parameters.

*Actions*: Similarly to the previous task, we need to add a schedule, which would be run once every month, we want to run it at the start of the day, so that all accruals from the previous day had a chance to be processed, recorded and would be included when we charge the account with interest. Again that would be implemented in the execution_schedules and scheduled_code hooks.

## Next Step

We are now ready to get started developing our smart contract. Clone this repo to a local directory, then run `git checkout exercise-1` to checkout the scripts and tutorial required.
