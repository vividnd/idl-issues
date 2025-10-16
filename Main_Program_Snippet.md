# **Main Program Snippet** - Critical Arcium Integration Logic

## **1. Callback Account Structure**
```rust
#[callback_accounts("survey_analytics")]
#[derive(Accounts)]
pub struct SurveyAnalyticsCallback<'info> {
    pub arcium_program: Program<'info, Arcium>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_SURVEY_ANALYTICS))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    /// CHECK: instructions_sysvar, checked by the account constraint
    #[account(address = ::anchor_lang::solana_program::sysvar::instructions::ID)]
    pub instructions_sysvar: AccountInfo<'info>,
    
    // ✅ FIX: Add callback accounts that are passed from queue_computation
    #[account(mut)]
    pub analytics_storage: Account<'info, SurveyAnalyticsStorage>,
    
    #[account(mut)]
    pub survey: Account<'info, Survey>,
}
```

## **2. Sign PDA Account Initialization**
```rust
pub fn init_sign_pda(ctx: Context<InitSignPda>) -> Result<()> {
    // Set the sign PDA account bump
    ctx.accounts.sign_pda_account.bump = ctx.bumps.sign_pda_account;
    Ok(())
}
```

## **3. Core Survey Analytics Function**
```rust
pub fn submit_survey_analytics(
    ctx: Context<SubmitSurveyAnalytics>,
    analytics_computation_offset: u64,    // For creator analytics
    // answer1: Enc<Shared, u32> (3 args)
    answer1_pub_key: [u8; 32],           // Encryption key for answer1
    answer1_nonce: u128,                 // Nonce for answer1
    ciphertext_answer1: [u8; 32],        // Encrypted answer1
    // answer2: Enc<Shared, u32> (3 args)
    answer2_pub_key: [u8; 32],           // Encryption key for answer2
    answer2_nonce: u128,                 // Nonce for answer2
    ciphertext_answer2: [u8; 32],        // Encrypted answer2
    // question_type1: Enc<Shared, u32> (3 args)
    question_type1_pub_key: [u8; 32],    // Encryption key for question_type1
    question_type1_nonce: u128,          // Nonce for question_type1
    ciphertext_question_type1: [u8; 32], // Encrypted question_type1
    // question_type2: Enc<Shared, u32> (3 args)
    question_type2_pub_key: [u8; 32],    // Encryption key for question_type2
    question_type2_nonce: u128,          // Nonce for question_type2
    ciphertext_question_type2: [u8; 32], // Encrypted question_type2
    // total_responses: Enc<Shared, u32> (3 args)
    total_responses_pub_key: [u8; 32],   // Encryption key for total_responses
    total_responses_nonce: u128,         // Nonce for total_responses
    ciphertext_total_responses: [u8; 32], // Encrypted total_responses
    // completion_rate: Enc<Shared, u32> (3 args)
    completion_rate_pub_key: [u8; 32],   // Encryption key for completion_rate
    completion_rate_nonce: u128,         // Nonce for completion_rate
    ciphertext_completion_rate: [u8; 32], // Encrypted completion_rate
    // survey_creator: Shared (2 args)
    survey_creator_pub_key: [u8; 32],    // survey_creator public key
    survey_creator_nonce: u128,          // survey_creator nonce
    // public_viewer: Shared (2 args)
    public_viewer_pub_key: [u8; 32],     // public_viewer public key
    public_viewer_nonce: u128,           // public_viewer nonce
    // respondent: Shared (2 args)
    respondent_pub_key: [u8; 32],        // respondent public key
    respondent_nonce: u128,              // respondent nonce
) -> Result<()> {
    // Set the sign PDA account bump
    ctx.accounts.sign_pda_account.bump = ctx.bumps.sign_pda_account;
    
    // Validate survey is active
    require!(ctx.accounts.survey.is_active, ErrorCode::SurveyInactive);
    
    // Check if survey has reached max responses
    require!(
        ctx.accounts.survey.current_responses < ctx.accounts.survey.max_responses,
        ErrorCode::SurveyFull
    );
```

## **4. Critical Argument Queuing Logic (24 Arguments)**
```rust
    // Computation 1: Analytics for survey creator
    // The survey_analytics function expects:
    // 1. 6 Enc<Shared, u32> = 18 args (3 args each: pubkey + nonce + encrypted_data)
    // 2. 3 Shared structs = 6 args (2 args each)
    // Total: 24 arguments
    let creator_args = vec![
        // answer1: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(answer1_pub_key),           // Encryption key for answer1
        Argument::PlaintextU128(answer1_nonce),           // Nonce for answer1
        Argument::EncryptedU32(ciphertext_answer1),       // Encrypted answer1
        // answer2: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(answer2_pub_key),           // Encryption key for answer2
        Argument::PlaintextU128(answer2_nonce),           // Nonce for answer2
        Argument::EncryptedU32(ciphertext_answer2),       // Encrypted answer2
        // question_type1: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(question_type1_pub_key),    // Encryption key for question_type1
        Argument::PlaintextU128(question_type1_nonce),    // Nonce for question_type1
        Argument::EncryptedU32(ciphertext_question_type1), // Encrypted question_type1
        // question_type2: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(question_type2_pub_key),    // Encryption key for question_type2
        Argument::PlaintextU128(question_type2_nonce),    // Nonce for question_type2
        Argument::EncryptedU32(ciphertext_question_type2), // Encrypted question_type2
        // total_responses: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(total_responses_pub_key),   // Encryption key for total_responses
        Argument::PlaintextU128(total_responses_nonce),   // Nonce for total_responses
        Argument::EncryptedU32(ciphertext_total_responses), // Encrypted total_responses
        // completion_rate: Enc<Shared, u32> (3 args)
        Argument::ArcisPubkey(completion_rate_pub_key),   // Encryption key for completion_rate
        Argument::PlaintextU128(completion_rate_nonce),   // Nonce for completion_rate
        Argument::EncryptedU32(ciphertext_completion_rate), // Encrypted completion_rate
        // survey_creator: Shared (2 args)
        Argument::ArcisPubkey(survey_creator_pub_key),    // survey_creator public key
        Argument::PlaintextU128(survey_creator_nonce),    // survey_creator nonce
        // public_viewer: Shared (2 args)
        Argument::ArcisPubkey(public_viewer_pub_key),     // public_viewer public key
        Argument::PlaintextU128(public_viewer_nonce),     // public_viewer nonce
        // respondent: Shared (2 args)
        Argument::ArcisPubkey(respondent_pub_key),        // respondent public key
        Argument::PlaintextU128(respondent_nonce),        // respondent nonce
    ];
```

## **5. Queue Computation Call with Callback**
```rust
    // Single computation for analytics
    queue_computation(
        ctx.accounts,
        analytics_computation_offset,
        creator_args,
        None,
        vec![SurveyAnalyticsCallback::callback_ix(&[
            CallbackAccount {
                pubkey: ctx.accounts.analytics_storage.key(),
                is_writable: true,
            },
            CallbackAccount {
                pubkey: ctx.accounts.survey.key(),
                is_writable: true,
            },
        ])],
    )?;

    // Increment response counter
    ctx.accounts.survey.current_responses += 1;

    // Emit event for tracking - avoid wallet exposure
    emit!(ResponseSubmitted {
        survey: ctx.accounts.survey.key(),
    });

    Ok(())
}
```

## **6. Account Structure for Queue Computation**
```rust
#[queue_computation_accounts("survey_analytics", payer)]
#[derive(Accounts)]
#[instruction(analytics_computation_offset: u64)]
pub struct SubmitSurveyAnalytics<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(
        init_if_needed,
        payer = payer,
        space = 9,
        seeds = [&SIGN_PDA_SEED],
        bump,
        address = derive_sign_pda!(),
    )]
    pub sign_pda_account: Account<'info, SignerAccount>,
    #[account(
        mut,
        address = derive_mxe_pda!()
    )]
    pub mxe_account: Box<Account<'info, MXEAccount>>,
    #[account(
        mut,
        address = derive_mempool_pda!()
    )]
    /// CHECK: mempool_account, checked by the arcium program.
    pub mempool_account: UncheckedAccount<'info>,
    #[account(
        mut,
        address = derive_execpool_pda!()
    )]
    /// CHECK: executing_pool, checked by the arcium program.
    pub executing_pool: UncheckedAccount<'info>,
    #[account(
        mut,
        address = derive_comp_pda!(analytics_computation_offset)
    )]
    /// CHECK: computation_account, checked by the arcium program.
    pub computation_account: UncheckedAccount<'info>,
    #[account(
        address = derive_comp_def_pda!(COMP_DEF_OFFSET_SURVEY_ANALYTICS)
    )]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    #[account(
        mut,
        address = derive_cluster_pda!(mxe_account)
    )]
    pub cluster_account: Account<'info, Cluster>,
    #[account(
        mut,
        address = ARCIUM_FEE_POOL_ACCOUNT_ADDRESS,
    )]
    pub pool_account: Account<'info, FeePool>,
    #[account(
        address = ARCIUM_CLOCK_ACCOUNT_ADDRESS
    )]
    pub clock_account: Account<'info, ClockAccount>,
    #[account(mut)]
    pub survey: Account<'info, Survey>,
    #[account(mut)]
    pub analytics_storage: Account<'info, SurveyAnalyticsStorage>,
    pub system_program: Program<'info, System>,
    pub arcium_program: Program<'info, Arcium>,
}
```

