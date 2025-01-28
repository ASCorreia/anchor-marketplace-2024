# NFT Marketplace

This example demonstrates how to create a simple NFT marketplace using the Token Metadata Program.

In this program, users can list their NFTs for sale, delist them, and purchase listed NFTs from other users. The marketplace includes features like marketplace fees and a treasury system.

---

## Let's walk through the architecture:

For this program, we will have 2 state accounts:

- A Marketplace account
- A Listing account

A Marketplace account consists of:

```rust
#[account]
pub struct Marketplace {
    pub admin: Pubkey,
    pub fee: u16,
    pub bump: u8,
    pub treasury_bump: u8,
    pub rewards_bump: u8,
    pub name: String, // Set the limit to 32 bytes
}
```

### In this state account, we will store:

- admin: The administrator's public key who manages the marketplace
- fee: The marketplace fee charged on each transaction
- bump: Since our Marketplace account will be a PDA, we store its bump
- treasury_bump: The bump for the marketplace's treasury PDA
- rewards_bump: The bump for the rewards mint PDA
- name: The marketplace identifier (limited to 32 bytes)

We implement the Space trait to calculate the amount of space our account will use on-chain including the anchor discriminator.

---

A Listing account consists of:

```rust
#[account]
pub struct Listing {
    pub maker: Pubkey,
    pub mint: Pubkey,
    pub price: u64,
    pub bump: u8,
}
```

### In this state account, we will store:

- maker: The public key of the user creating the listing
- mint: The public key of the NFT being listed
- price: The price in lamports that the NFT is listed for
- bump: Since our Listing account will be a PDA, we store its bump

Similar to the Marketplace, the Space trait is implemented.

---

### An admin can initialize a new marketplace. For that, we create the following context:

```rust
#[derive(Accounts)]
#[instruction(name: String)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub admin: Signer<'info>,
    #[account(
        init,
        payer = admin,
        seeds = [b"marketplace", name.as_str().as_bytes()],
        bump,
        space = Marketplace::INIT_SPACE
    )]
    pub marketplace: Account<'info, Marketplace>,
    #[account(
        seeds = [b"treasury", marketplace.key().as_ref()],
        bump,
    )]
    pub treasury: SystemAccount<'info>,
    #[account(
        init,
        payer = admin,
        seeds = [b"rewards", marketplace.key().as_ref()],
        bump,
        mint::decimals = 6,
        mint::authority = marketplace,
    )]
    pub rewards_mint: InterfaceAccount<'info, Mint>,
    pub system_program: Program<'info, System>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

Let's break down the accounts needed for initialization:

- `admin`: The marketplace administrator's account. Must be a signer and mutable since they'll pay for account initialization fees.
- `marketplace`: A PDA account derived from [b"marketplace", name.as_str().as_bytes()]. This account stores the marketplace configuration and is initialized by this instruction.
- `treasury`: A PDA account derived from [b"treasury", marketplace.key()]. This account will collect marketplace fees from sales.
- `rewards_mint`: A PDA token mint derived from [b"rewards", marketplace.key()]. Initialized with 6 decimals and the marketplace as the mint authority.
- `system_program`: Required for creating new accounts.
- `token_program`: Required for creating the rewards mint.

The initialization process creates all necessary PDAs and sets up the marketplace configuration with the provided name and fee structure.

Note: You can access the instruction’s arguments with the `#[instruction(..)]` attribute. Learn more [here](https://www.anchor-lang.com/docs/account-constraints#instruction-attribute)

---

### Users can list their NFTs for sale using the List instruction:

```rust
#[derive(Accounts)]
pub struct List<'info> {
    #[account(mut)]
    pub maker: Signer<'info>,
    #[account(
        seeds = [b"marketplace", marketplace.name.as_str().as_bytes()],
        bump = marketplace.bump,
    )]
    pub marketplace: Account<'info, Marketplace>,
    pub maker_mint: InterfaceAccount<'info, Mint>,
    #[account(
        mut,
        associated_token::mint = maker_mint,
        associated_token::authority = maker,
    )]
    pub maker_ata: InterfaceAccount<'info, TokenAccount>,
    #[account(
        init,
        payer = maker,
        associated_token::mint = maker_mint,
        associated_token::authority = listing,
    )]
    pub vault: InterfaceAccount<'info, TokenAccount>,
    #[account(
        init,
        payer = maker,
        seeds = [marketplace.key().as_ref(), maker_mint.key().as_ref()],
        bump,
        space = Listing::INIT_SPACE,
    )]
    pub listing: Account<'info, Listing>,
    pub collection_mint: InterfaceAccount<'info, Mint>,
    #[account(
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            maker_mint.key().as_ref(),
        ],
        seeds::program = metadata_program.key(),
        bump,
        constraint = metadata.collection.as_ref().unwrap().key.as_ref() == collection_mint.key().as_ref(),
        constraint = metadata.collection.as_ref().unwrap().verified == true,
    )]
    pub metadata: Account<'info, MetadataAccount>,
    #[account(
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            maker_mint.key().as_ref(),
            b"edition"
        ],
        seeds::program = metadata_program.key(),
        bump,
    )]
    pub master_edition: Account<'info, MasterEditionAccount>,
    pub metadata_program: Program<'info, Metadata>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

Let's break down the accounts needed for listing an NFT:

- `maker`: The user listing the NFT. Must be a signer and mutable as they'll pay for account creation.
- `marketplace`: The marketplace PDA account containing configuration. Used to verify marketplace parameters.
- `maker_mint`: The token mint account of the NFT being listed.
- `maker_ata`: The maker's Associated Token Account holding the NFT. Must be mutable for NFT transfer.
- `vault`: A new Associated Token Account that will hold the NFT during the listing. Authority is the listing PDA.
- `listing`: A new PDA account derived from [marketplace.key(), maker_mint.key()]. Stores listing information.
- `collection_mint`: The mint account of the NFT collection.
- `metadata`: The NFT metadata account, used to verify the NFT belongs to the correct collection.
- `master_edition`: The master edition account of the NFT.
- `metadata_program`: Required for verifying NFT metadata.
- `associated_token_program`: Required for creating the vault ATA.
- `system_program`: Required for creating the listing account.
- `token_program`: Required for NFT transfer operations.

The listing process:
1. Verifies the NFT belongs to a verified collection
2. Creates a new listing PDA to store sale information
3. Creates a vault account owned by the listing PDA
4. Transfers the NFT from the maker's ATA to the vault

---

### Users can delist their NFTs using the Delist instruction:

```rust
#[derive(Accounts)]
pub struct Delist<'info> {
    #[account(mut)]
    maker: Signer<'info>,
    marketplace: Account<'info, Marketplace>,
    maker_mint: InterfaceAccount<'info, Mint>,
    #[account(mut)]
    maker_ata: InterfaceAccount<'info, TokenAccount>,
    #[account(mut)]
    listing: Account<'info, Listing>,
    #[account(mut)]
    vault: InterfaceAccount<'info, TokenAccount>,
}
```

Let's break down the accounts needed for delisting an NFT:

- `maker`: The original lister of the NFT. Must be a signer to authorize delisting.
- `marketplace`: The marketplace PDA account, used to verify marketplace context.
- `maker_mint`: The token mint account of the NFT being delisted.
- `maker_ata`: The maker's Associated Token Account that will receive back the NFT. Must be mutable.
- `listing`: The listing PDA account that will be closed, with rent returned to maker.
- `vault`: The vault Associated Token Account holding the NFT, which will be emptied and closed.
- `token_program`: Required for NFT transfer operations.
- `system_program`: Required for closing accounts and rent refunds.

The delisting process:
1. Verifies the maker is the original lister
2. Transfers the NFT from the vault back to the maker's ATA
3. Closes the vault account and returns rent to the maker
4. Closes the listing account and returns rent to the maker

---

### Users can purchase listed NFTs using the Purchase instruction:

```rust
#[derive(Accounts)]
pub struct Purchase<'info> {
    #[account(mut)]
    pub taker: Signer<'info>,
    #[account(mut)]
    pub maker: SystemAccount<'info>,
    pub maker_mint: InterfaceAccount<'info, Mint>,
    pub marketplace: Account<'info, Marketplace>,
    #[account(mut)]
    pub taker_ata: InterfaceAccount<'info, TokenAccount>,
    #[account(mut)]
    pub vault: InterfaceAccount<'info, TokenAccount>,
    #[account(mut)]
    pub listing: Account<'info, Listing>,
    pub treasury: SystemAccount<'info>,
    pub rewards_mint: InterfaceAccount<'info, Mint>,
}
```

Let's break down the accounts needed for purchasing a listed NFT:

- `taker`: The buyer's account. Must be a signer and mutable as they'll pay for the NFT and any init fees.
- `maker`: The seller's account. Mutable to receive payment.
- `maker_mint`: The token mint account of the NFT being purchased.
- `marketplace`: The marketplace PDA account containing fee configuration.
- `taker_ata`: The buyer's Associated Token Account that will receive the NFT. Created if it doesn't exist.
- `vault`: The vault Associated Token Account currently holding the NFT.
- `listing`: The listing PDA account that will be closed after the sale.
- `treasury`: The marketplace treasury PDA that receives the marketplace fee.
- `rewards_mint`: The marketplace rewards mint PDA for potential reward token distribution.
- `associated_token_program`: Required for creating the buyer's ATA if needed.
- `token_program`: Required for NFT transfer operations.
- `system_program`: Required for SOL transfers and account management.

The purchase process:
1. Creates the buyer's ATA if it doesn't exist
2. Calculates and transfers the marketplace fee to the treasury
3. Transfers the remaining payment to the seller
4. Transfers the NFT from the vault to the buyer's ATA
5. Closes the vault account and returns rent to the seller
6. Closes the listing account and returns rent to the seller

Each instruction includes validation and ensures that only authorized users can perform actions on their own listings. The marketplace maintains control through PDAs and account validations.

---

## Program Structure and Entry Points

The program exposes four main entry points in the `anchor_marketplace` module:

1. `initialize(ctx: Context<Initialize>, name: String, fee: u16)`:
   - Creates a new marketplace instance with the specified name and fee structure
   - Sets up the treasury and rewards system

2. `listing(ctx: Context<List>, price: u64)`:
   - Creates a new NFT listing with the specified price
   - Transfers the NFT to the vault

3. `delist(ctx: Context<Delist>)`:
   - Removes an NFT listing
   - Returns the NFT to the original owner

4. `purchase(ctx: Context<Purchase>)`:
   - Processes the purchase of a listed NFT
   - Handles the SOL transfer, including marketplace fees
   - Transfers the NFT to the buyer
   - Cleans up the listing accounts
