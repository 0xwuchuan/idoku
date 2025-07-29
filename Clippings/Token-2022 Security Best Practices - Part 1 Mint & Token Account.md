---
title: "Token-2022 Security Best Practices - Part 1: Mint & Token Account"
source: "https://blog.offside.io/p/token-2022-security-best-practices-part-1"
author:
  - "[[Offside Labs]]"
published: 2024-09-19
created: 2025-07-23
description: "This is the first article in the Token-2022 Security Best Practices series. It discusses potential security vulnerabilities in Mint and Token Accounts during Solana development when supporting Token-2022 and offers recommended design and implementation strategies."
tags:
  - "clippings"
---
![](https://substackcdn.com/image/fetch/$s_!rY1y!)

## Introduction

As ***[Solana](https://solana.com/)*** continues to evolve, users' demands for sophisticated and flexible token functionalities are growing. However, ***[Token](https://spl.solana.com/token)*** on Solana have so far only provided basic functionalities, which are unable to meet this increasing demand.

In response, Solana has introduced ***[Token-2022](https://spl.solana.com/token-2022)***, a suite of enhanced features designed to expand token capabilities within the ecosystem. Token-2022 is fully compatible with the existing SPL Token and extends native token features through nearly 20 Token Extensions.

There is not much publicly available material on the types of security issues. The purpose of this series of articles is to shed light on the secure development, helping Solana developers avoid potential security risks while implementing support for Token-2022.

This first article of our series will discuss the following 3 topics. And we hope you can pay extra attention to the ***Attention*** section in each topic.

- Security risks across various stages of Account Lifetime
- Commonly confused terms between SPL Token and Token-2022
- Best practices for securely using Token-2022 in Anchor

## How to secure accounts during its whole lifetime?

Following the design principles of `Solana Account Model`, the extensions in Token-2022 are built around the `Token Account` and `Mint Account`. Throughout the various stages of these Account Lifetimes, there exist several security risks to which developers must pay attention during both the design and implementation phases. We will illustrate these risks for `Token Account` and `Mint Account` respectively.

## 1\. Token Account

As for the Token Accounts, the essential difference between Token-2022 and SPL Token is the addition of the extension data.

In terms of implementation, Token-2022 serves as a proper superset of SPL Token, maintaining the structural layout of Token Accounts while appending additional fields of Extension Data to the original Token Account.

The general structure is depicted as follows.

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/db88af2a-fd7c-4846-a6ce-2364c64ddd4f_558x1151.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:1151,%22width%22:558,%22resizeWidth%22:180,%22bytes%22:74059,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

Here is a list with the extensions that could be appended to Token Account：

- Immutable Owner
- CPI Guard
- Required Memo on Transfer
- Non-Transferable Tokens
- Transfer Fees
- Transfer Hook
- Confidential Transfer
- Confidential Transfer Fee

We will examine the three stages of the Token Account lifecycle: creation, reallocation, and closure.

At each stage, there are potential security risks that developers should be aware of.

### 1.1. The Creation of Token Account

The size of Token Account in Token-2022 ***CAN*** differ from that of SPL Token, depending on the types and number of Extensions supported by the Mint and Token Account. Different sizes of Token Accounts will be created based on the number of Extensions supported. These accounts can take up more space than those in SPL Token, and their size will grow as additional extensions are added.

#### Attention

> It is `not recommended` to `hardcode a fixed space size ` for the Token Account or the minimum rent-exempt amount that the fixed space size requires, if you need to create a Token-2022 Token Account for a user at the runtime.

#### Vulnerable Case

The function below allows a Keeper to dynamically create a Token Account with Token-2022 support.

```markup
export const getCostToOpenTokenAccount = async () => {
  const cost = 0.00203928 * (10 ** 9);
  return cost;
};
```

A hardcoded SOL amount of `0.00203928` is used when creating the Token Account, which matches the rent needed for the *165 bytes* required by a Token Account in the Token Program.

The problem arises when this Token Account needs to support Token-2022 extensions: the specified amount is *insufficient* to cover the extra space needed by Token-2022 account extensions, then the creation of Token Account will fail.

If the space size is determined by the user and the Backend Keeper is responsible for creating the Token Account, there will be a potential risk of the Keeper overpaying rent, resulting in financial losses.

Therefore, it is recommended to avoid having the Keeper create Token Accounts for users.

#### Recommendation

The `@solana/spl-token` library now offers API interfaces that assist in calculating the account length and the minimum rent requirement with extensions.

- `getExtensionTypes`: Returns all the extensions supported by the mint.
- `getMinimumBalanceForRentExemptAccountWithExtensions`: Calculates the minimum rent needed for a token account with extensions.

Here is an improved code snippet to mitigate the above issue.

```markup
export const getCostToOpenTokenAccount = async (mint) => {
    const extensions = getExtensionTypes(mint.tlvData);
    const rent = await getMinimumBalanceForRentExemptAccountWithExtensions(
      connection,
      [...extensions]
    );
    const cost = rent * (10 ** mint.decimals);
    return cost;
}
```

### 1.2 Reallocation

Some Account Extensions, such as Memo Transfer, can be added to a Token Account after its initial creation. Thus, the size of a Token Account may change dynamically.

To support the addition of new Account Extensions after the creation of Token Account, Token-2022 introduces a `Reallocate` IX that allows increasing the size of an existing Token Account to accommodate additional extension bytes.

> Within the Reallocate IX, if it is determined that the current Token Account size already meets the necessary requirements, the process will return without further reallocation.

#### Attention

> When reallocating a Token Account to a larger size, additional rent will be charged according to the number of additional bytes. It is essential to consider who will cover this extra rent during the development process, as overlooking this aspect could lead to the protocol or project covering unexpected costs.

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/a2b8b8db-faad-40ba-895a-15d54d4967cb_1971x1730.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:1278,%22width%22:1456,%22resizeWidth%22:510,%22bytes%22:259839,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

> In particular, for the APIs that may be used, it is important to carefully consider how to set values for these parameters.
> 
> - `payer` parameter in [createReallocateInstruction](https://solana-labs.github.io/solana-program-library/token/js/functions/createReallocateInstruction.html)
> - `payer` parameter in [reallocate](https://docs.rs/spl-token-2022/4.0.1/src/spl_token_2022/instruction.rs.html#1848)

### 1.3. Closing the Token Account

A Token Account can be closed.

In SPL Token, a `non-WSOL` Token Account can be closed by the `account.authority` as long as `account.amount == 0` is satisfied.

However, in Token-2022, some additional conditions must be met to close a Token Account:

- `account.amount == 0`
- If the close method is invoked in a CPI context and the Token Account has the `CPI Guard` extension enabled, the destination for the lamports must be the owner of Token Account.
- If the Token Account supports the `Confidential Transfer` extension, both `pending_balance == 0` and `available_balance == 0` must be true.
- If the Token Account supports the `Confidential Transfer Fee` extension, `withheld_amount == 0` must be ensured.
- If the Token Account supports the `Transfer Fee` extension, `withheld_amount == 0` must be ensured.

This is a simplified check flow in [SPL](https://github.com/solana-labs/solana-program-library/blob/token-2022-v4.0.1/token/program-2022/src/processor.rs#L1134-L1181):

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/87d1fee6-75d0-4108-b268-2786b27d6606_5097x1098.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:314,%22width%22:1456,%22resizeWidth%22:null,%22bytes%22:335897,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

#### Attention

> If the contracts need to determine at runtime whether the criteria for closing a Token Account in Token-2022 are fulfilled, it is imperative that all the aforementioned conditions are satisfied. Otherwise, the Token Account cannot be closed directly.

#### Vulnerable Case

In the following contract code snippet, an attempt is made to close Token Account A when its amount is 0.

```markup
if ctx.accounts.A.amount == 0 {
            close_account(CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                CloseAccount {
                    account: ctx.accounts.A.to_account_info(),
                    destination: ctx.accounts.B.to_account_info(),
                    authority: ctx.accounts.B.to_account_info(),
                },
                signer_seeds,
            ))?;
        }
```

If A is a Token Account of Token-2022 that supports Transfer Fee extensions, and `TransferFeeAmount.withheld_amount` is larger than 0, then it cannot be closed and an error will be raised in `close_account`: `AccountHasWithheldTransferFees: "An account can only be closed if its withheld fee balance is zero, harvest fees to the mint and try again"`. This error will cause the contract to fail to execute the IX because the condition `TransferFeeAmount.withheld_amount == 0` was not checked when verifying if A meets the closing requirements.

The `TransferFeeAmount` extension provides `closable` method to check this condition to avoid raising the error. Similarly, the other extensions also provide `closable` method for checking.

#### Recommendation

The following examples show how to close Token Accounts with various extensions and illustrate how to verify whether these extensions are closable.

##### Cpi Guard

```markup
use solana_program::instruction::{get_stack_height, TRANSACTION_LEVEL_STACK_HEIGHT};
...
     if let Ok(cpi_guard) = source_account.get_extension::<CpiGuard>() {
         if cpi_guard.lock_cpi.into()
             && get_stack_height() > TRANSACTION_LEVEL_STACK_HEIGHT
             && !cmp_pubkeys(destination_account_info.key, &source_account.base.owner)
         {
             return Err(TokenError::CpiGuardCloseAccountBlocked.into());
         }
     }
```

##### Transfer Fee

```markup
if let Ok(transfer_fee_state) = source_account.get_extension::<TransferFeeAmount>() {
       transfer_fee_state.closable()?
   }
```

##### Confidential Transfer

```markup
if let Ok(confidential_transfer_state) =
       source_account.get_extension::<ConfidentialTransferAccount>()
   {
       confidential_transfer_state.closable()?
   }
```

##### Confidential Transfer Fee

```markup
if let Ok(confidential_transfer_fee_state) =
       source_account.get_extension::<ConfidentialTransferFeeAmount>()
   {
       confidential_transfer_fee_state.closable()?
   }
```

You can refer to the links below for detailed code implementations.

- Transfer Fee
	- TransferFeeAmount: [closable()](https://github.com/solana-labs/solana-program-library/blob/3522730070da7b7b6c19513b288c9faebd0be2ab/token/program-2022/src/extension/transfer_fee/mod.rs#L183)
- Confidential Transfer
	- ConfidentialTransferAccount: [closable()](https://github.com/solana-labs/solana-program-library/blob/3522730070da7b7b6c19513b288c9faebd0be2ab/token/program-2022/src/extension/confidential_transfer/mod.rs#L136)
- Confidential Transfer Fee
	- ConfidentialTransferFeeAmount: [closable()](https://github.com/solana-labs/solana-program-library/blob/3522730070da7b7b6c19513b288c9faebd0be2ab/token/program-2022/src/extension/confidential_transfer_fee/mod.rs#L70)

## 2\. Mint Account

Similar to Token Accounts, Token-2022 maintains the structure layout of the Mint Account and appends additional Extension Data fields.

Layout diagram is provided below.

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/fde6ab79-abec-4ab7-973c-78beee61c7df_558x1151.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:1151,%22width%22:558,%22resizeWidth%22:166,%22bytes%22:72009,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

Here is a list with the extensions that could be appended to Mint Account：

- Non-Transferable Tokens
- Transfer Fees
- Transfer Hook
- Confidential Transfer
- Confidential Transfer Fee
- Mint Close Authority
- Default Account State
- Interest-Bearing Tokens
- Permanent Delegate
- Metadata Pointer
- Metadata
- Group Pointer
- Group
- Group Member Pointer
- Group Member

Next, let's examine the security risks associated with the Mint Account at its creation and termination stages that developers should be aware of.

### 2.1. Create Mint

Different from Token Accounts, Token-2022 requires all extensions to be created before initializing the Mint Account. This is because most Mint Extensions are closely related to the Mint's tokenomics and core functionality.

Here is the list of Mint Extensions:

- confidential transfer
- confidential transfer fee
- default account state
- group member pointer
- group pointer
- metadata pointer
- interest-bearing mint
- Transfer fee
- Transfer hook
- Immutable Owner
- Mint Close Authority
- Non Transferable
- Permanent Delegate

Furthermore, when initializing the Mint Account, [additional checks](https://github.com/solana-labs/solana-program-library/blob/token-2022-v4.0.1/token/program-2022/src/extension/mod.rs#L1292-L1321) are required due to predefined constraints among Mint extensions. For example:

- If the `confidential transfer fee` is enabled, the `transfer fee` and `confidential transfer` must also be enabled.
- If the `transfer fee` and `confidential transfer` are enabled, the `confidential transfer fee` must also be enabled.

#### Attention

> Unlike Token Accounts, where extensions can be added after creation, all extensions for a Mint Account must be initialized before the Mint is created. Once a Mint Account is created, it is not possible to add new extensions.
> 
> This code snippet illustrates how to create a Mint with extensions.
> 
> ```markup
> // List extensions to be added to the new mint
>     const extensions = [ExtensionType.TransferFeeConfig];
> 
>     // Calculate length of mint
>     const mintLen = getMintLen(extensions);
> 
>     // Transfer Fee Config Parameters
>     const decimals = 9;
>     const feeBasisPoints = 50;
>     const maxFee = BigInt(5_000);
>     // Calculate minimum rent for create the new mint account
>     const mintLamports = await connection.getMinimumBalanceForRentExemption(mintLen);
> 
>     // Mint Creation IXs
>     // 1. Create a raw mint account with calculated mintLen and mintLamports
>     // 2. Create TransferFee Config and append this extension to the mint account
>     // 3. Initialize Mint 
>     const mintTransaction = new Transaction().add(
>         SystemProgram.createAccount({
>             fromPubkey: payer.publicKey,
>             newAccountPubkey: mint,
>             space: mintLen,
>             lamports: mintLamports,
>             programId: TOKEN_2022_PROGRAM_ID,
>         }),
>         createInitializeTransferFeeConfigInstruction(
>             mint,
>             transferFeeConfigAuthority.publicKey,
>             withdrawWithheldAuthority.publicKey,
>             feeBasisPoints,
>             maxFee,
>             TOKEN_2022_PROGRAM_ID
>         ),
>         createInitializeMintInstruction(mint, decimals, mintAuthority.publicKey, null, TOKEN_2022_PROGRAM_ID)
>     );
>     await sendAndConfirmTransaction(connection, mintTransaction, [payer, mintKeypair], undefined);
> ```

### 2.2. Close Mint

In SPL Token, Mint accounts cannot be closed.

However, in Token-2022, it is possible to close a Mint by applying the `MintCloseAuthority` extension.

#### Attention

> `MintCloseAuthority` requires that `Mint.supply == 0`.
> 
> When the Mint is `closable`, the amount of all token accounts related to this mint should be **zero** at that time.
> 
> However, if the Mint is closed and re-created with a different purpose at the same address. Inconsistencies will arise if your project stores information related to the Mint Account. If your project stores mint information, be aware of this potential issue and consider redesigning the code to ensure that it consistently fetches data from the correct mint.

## 3\. Summary

Some extensions in Token-2022 add data only to the Mint Account, some only to the Token Account, and others append data to both the Mint Account and the Token Account.

At the end of this section, we offer a brief summary that correlates these extensions with their respective accounts.

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/20d2f82e-a955-4b63-8e68-2d10d1feab7c_550x812.jpeg%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:812,%22width%22:550,%22resizeWidth%22:null,%22bytes%22:116587,%22alt%22:null,%22title%22:null,%22type%22:%22image/jpeg%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

## Token Twins: Are You Using the Right One?

Token-2022, being a proper superset of SPL Token, fully supports all 25 Token IXs with the same instruction layout and identical format.

In the following section, we outline several key differences between SPL Token and Token-2022 in the SPL.

## transfer VS transfer\_checked

In SPL Token, the `spl_token::instruction::transfer` method is used for mint transfer. However, in Token-2022, the `transfer` method has been `deprecated`.

```markup
/// Creates a \`Transfer\` instruction.
#[deprecated(
    since = "4.0.0",
    note = "please use \`transfer_checked\` or \`transfer_checked_with_fee\` instead"
)]
pub fn transfer(
```

The above code comments clearly indicate that it is recommended to use `transfer_checked` or `transfer_checked_with_fee`.

So, what issues might arise if you still uses the transfer method?

First, let’s review the definitions of the two suggested functions:

```markup
/// Creates a \`TransferChecked\` instruction.
#[allow(clippy::too_many_arguments)]
pub fn transfer_checked(
    token_program_id: &Pubkey,
    source_pubkey: &Pubkey,
    mint_pubkey: &Pubkey,
    destination_pubkey: &Pubkey,
    authority_pubkey: &Pubkey,
    signer_pubkeys: &[&Pubkey],
    amount: u64,
    decimals: u8,
) -> Result<Instruction, ProgramError> {
...
}

/// Create a \`TransferCheckedWithFee\` instruction
#[allow(clippy::too_many_arguments)]
pub fn transfer_checked_with_fee(
    token_program_id: &Pubkey,
    source: &Pubkey,
    mint: &Pubkey,
    destination: &Pubkey,
    authority: &Pubkey,
    signers: &[&Pubkey],
    amount: u64,
    decimals: u8,
    fee: u64,
) -> Result<Instruction, ProgramError> {
    ...
}
```

The changes could be found from diffs in `transfer vs transfer_checked` and `transfer vs transfer_checked_with_fee`.

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/ecccbb32-94b1-4f79-a8be-fa0103f11f3b_1528x372.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:354,%22width%22:1456,%22resizeWidth%22:null,%22bytes%22:83282,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/8361b728-c6d1-40d2-9afa-41193902e3ee_1478x404.png%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:398,%22width%22:1456,%22resizeWidth%22:null,%22bytes%22:105796,%22alt%22:null,%22title%22:null,%22type%22:%22image/png%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

From the diff result, we have the following observations:

- `transfer_checked` has 2 additional parameters compared to transfer: `mint_pubkey` and `decimals`.
- `transfer_checked_with_fee` has 3 additional parameters compared to `transfer`: `mint`, `decimals`, and `fee`.

By reviewing the implementation of the transfer IX in Token-2022 program, it becomes clear that use of transfer may result in failure if the Token Account supports the `Transfer Hook` or `Transfer Fees` extensions, as these would prevent the transfer from being completed.

```markup
/// Processes a [Transfer](enum.TokenInstruction.html) instruction.
    pub fn process_transfer(
        program_id: &Pubkey,
        accounts: &[AccountInfo],
        amount: u64,
        expected_decimals: Option<u8>,
        expected_fee: Option<u64>,
    ) -> ProgramResult {
        ...
        let expected_mint_info = if let Some(expected_decimals) = expected_decimals {
            Some((next_account_info(account_info_iter)?, expected_decimals))
        } else {
            None
        };
        ...
        let (fee, maybe_permanent_delegate, maybe_transfer_hook_program_id) =
            if let Some((mint_info, expected_decimals)) = expected_mint_info {
                ...
            } else {
                // Transfer hook extension exists on the account, but no mint
                // was provided to figure out required accounts, abort
                if source_account
                    .get_extension::<TransferHookAccount>()
                    .is_ok()
                {
                    return Err(TokenError::MintRequiredForTransfer.into());
                }
                // Transfer fee amount extension exists on the account, but no mint
                // was provided to calculate the fee, abort
                if source_account
                    .get_extension_mut::<TransferFeeAmount>()
                    .is_ok()
                {
                    return Err(TokenError::MintRequiredForTransfer.into());
                } else {
                    (0, None, None)
                }
            };
        ...
```

The code shows that when the `transfer` function is invoked with `expected_decimals` set to `None`, it triggers the `else` branch in the code snippet. If the Token Account supports `Transfer Hook` or `Transfer Fees`, the `transfer` function will immediately return a `MintRequiredForTransfer` error and exit.

#### Attention

> It is recommended to stop using `transfer` in Token-2022 and use `transfer_checked` and `transfer_checked_with_fee` instead.
> 
> The full rust crate path for methods:
> 
> - transfer\_checked: `spl_token::instruction::transfer_checked`
> - transfer\_checked\_with\_fee: `spl_token_2022::extension::transfer_fee::instruction::transfer_checked_with_fee`

## Two wSOLs

In SPL Token, the account ID for WSOL is `So11111111111111111111111111111111111111112`.

Token-2022 has introduced a new WSOL with the account ID `9pan9bMn5HatX4EJdBwg9VgCa7Uz5HL8N1m5D3NdXejP`.

#### Attention

> If your contract needs to special handling WSOL, it is important to distinguish WSOL between SPL Token and Token-2022.
> 
> As of September 2024, the WSOL in Token-2022 has very little trading activity, so WSOL usually refers to the SPL Token's `So11111111111111111111111111111111111111112`.
> 
> For some DeFi platforms, where SOL/WSOL assets carry specific importance, it is advisable to blacklist the Token-2022 WSOL address to avoid any potential ambiguity.

## The Different Program ID

SPL Token Program and Token-2022 Program are separate programs, each with a different Program ID.

- SPL Token Program ID: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- Token-2022 Program ID: `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`

#### Attention

> In some Solana SDK interfaces, if the corresponding instructions work for both SPL Token and Token-2022, the SPL Token program is typically used as the default program ID parameter.  
> For example, here's the declaration of the `createAssociatedTokenAccountInstruction` function in the `solana/@spl_token` SDK:
> 
> ```markup
> export function createAssociatedTokenAccountInstruction(
>     payer: PublicKey,
>     associatedToken: PublicKey,
>     owner: PublicKey,
>     mint: PublicKey,
>     programId = TOKEN_PROGRAM_ID,       // <---- SPL Token Program ID
>     associatedTokenProgramId = ASSOCIATED_TOKEN_PROGRAM_ID
> )
> ```
> 
> When using these SDK interfaces, developers should `clearly identify which Token program they need to call`.

## How to Integrate Token-2022 into an Anchor Project?

[Anchor](https://www.anchor-lang.com/), the most popular development framework on Solana, supports Token-2022 by providing various types and IX wrappers. Developers could utilize these types and methods provided by Anchor.

## Types

Here is a summary of the type definitions and the SPL Token Program types they support:

![](https://blog.offside.io/p/%7B%22src%22:%22https://substack-post-media.s3.amazonaws.com/public/images/5113f903-c2cc-4669-b525-1198ee62bd26_731x429.jpeg%22,%22srcNoWatermark%22:null,%22fullscreen%22:null,%22imageSize%22:null,%22height%22:429,%22width%22:731,%22resizeWidth%22:null,%22bytes%22:66374,%22alt%22:null,%22title%22:null,%22type%22:%22image/jpeg%22,%22href%22:null,%22belowTheFold%22:true,%22topImage%22:false,%22internalRedirect%22:null,%22isProcessing%22:false,%22align%22:null,%22offset%22:false%7D)

We have some observations from the table:

- Types under the `token_interface` path are compatible with both the SPL Token Program and the Token-2022 Program.
- Types under the `token` path are only compatible with the SPL Token Program.
- Types under the `token_2022` path are exclusively for the Token-2022 Program.

#### Attention

> When developing, it is crucial to decide whether the contract needs to support Token-2022.
> 
> - If Token-2022 support is `required`, it is recommended to use the types under `anchor_spl::token_interface`.
> - If Token-2022 support is `not needed`, it is recommended to use the types under `anchor_spl::token::Token`.
> 
> If your contract does not intend to support Token-2022 but uses types from the `anchor_spl::token_interface` path, it could lead to unexpected issues due to ambiguity.

## Conclusion

This is our first article on Token-2022 series, based on our experiences from dozens of Token-2022-related audits. The article highlights potential security risks in Token-2022 accounts during development. Developers should be cautious, as improper design or implementation can lead to significant threats to project security.

In the next article, we will dive deeper into `Extensions` in Token-2022 that could affect the security of funds for both projects and users. Stay tuned for more insights.