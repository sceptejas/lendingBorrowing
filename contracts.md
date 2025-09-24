# 📋 QUBE-main Smart Contracts Documentation

## 🎯 Overview

The QUBE-main contracts implement a comprehensive Bitcoin lending platform on the Aptos blockchain using the Move programming language. The system allows users to deposit Bitcoin as collateral and borrow against it while earning yield through a sophisticated tokenization mechanism.

## 🏗️ System Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Collateral     │◄──►│   Loan Manager   │◄──►│ Interest Rate   │
│     Vault       │    │   (Central Hub)  │    │     Model       │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   ctrlBTC       │    │     lnBTC        │    │     xBTC        │
│    Token        │    │     Token        │    │   (Test Token)  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## 📄 Contract Details

### 1. 🏦 Collateral Vault (`collateral_vault.move`)

**Purpose**: Secure storage and management of user collateral for the BTC lending platform.

**Key Features**:
- **Collateral Management**: Tracks total and locked collateral per user
- **Security Controls**: Emergency pause functionality and admin controls
- **Integration**: Coordinates with ctrlBTC token for collateral representation

**Main Functions**:
- `deposit_collateral()` - Deposit BTC and receive ctrlBTC tokens
- `withdraw_collateral()` - Withdraw available (unlocked) collateral
- `lock_collateral()` - Lock collateral for active loans (LoanManager only)
- `unlock_collateral()` - Release collateral after loan repayment
- `get_user_available_collateral()` - Check withdrawable collateral

**Access Control**:
- **Admin**: Can pause/unpause, update addresses, transfer admin rights
- **LoanManager**: Can lock/unlock collateral for loans
- **Users**: Can deposit/withdraw their own collateral

---

### 2. 🎫 ctrlBTC Token (`ctrl_btc_token.move`)

**Purpose**: ERC20-style token representing BTC deposited as collateral in the lending system.

**Key Features**:
- **1:1 Backing**: Each ctrlBTC represents 1 unit of deposited BTC collateral
- **Controlled Minting**: Only CollateralVault can mint/burn tokens
- **Standard Transfers**: Users can transfer ctrlBTC tokens freely

**Main Functions**:
- `mint()` - Create new ctrlBTC tokens (CollateralVault only)
- `burn()` - Destroy ctrlBTC tokens (CollateralVault only)
- `transfer()` - Standard token transfer between users
- `balance()` - Check token balance for an address

**Token Details**:
- **Name**: "Collateral BTC"
- **Symbol**: "ctrlBTC"
- **Decimals**: 8 (same as Bitcoin)

---

### 3. 📊 Interest Rate Model (`interest_rate_model.move`)

**Purpose**: Manages dynamic interest rates based on Loan-to-Value (LTV) ratios.

**Key Features**:
- **Risk-Based Pricing**: Higher LTV ratios result in higher interest rates
- **Configurable Rates**: Admin can set custom rates for different LTV levels
- **Conservative Fallback**: Uses nearest higher LTV rate for undefined ratios

**Default Rate Structure**:
- **30% LTV**: 5% interest (500 basis points)
- **45% LTV**: 8% interest (800 basis points)
- **60% LTV**: 10% interest (1000 basis points)

**Main Functions**:
- `get_rate()` - Retrieve interest rate for a given LTV ratio
- `set_rate()` - Update interest rate for specific LTV (admin only)
- `get_all_ltv_ratios()` - List all configured LTV ratios
- `remove_rate()` - Remove rate for specific LTV (admin only)

**Constraints**:
- **Maximum LTV**: 60%
- **Maximum Interest Rate**: 50% (5000 basis points)

---

### 4. 💰 lnBTC Token (`ln_btc_token.move`)

**Purpose**: Represents loan BTC issued to borrowers in the lending system.

**Key Features**:
- **Loan Representation**: Each lnBTC represents 1 unit of borrowed BTC
- **Controlled Supply**: Only LoanManager can mint/burn tokens
- **Repayment Mechanism**: Tokens are burned when loans are repaid

**Main Functions**:
- `mint()` - Create new lnBTC tokens for loans (LoanManager only)
- `burn()` - Destroy lnBTC tokens for repayments (LoanManager only)
- `transfer()` - Standard token transfer between users
- `withdraw()` - Helper function for token burning

**Token Details**:
- **Name**: "Loan BTC"
- **Symbol**: "lnBTC"
- **Decimals**: 8 (same as Bitcoin)

---

### 5. 🎛️ Loan Manager (`loan_manager.move`)

**Purpose**: Central orchestrator managing the complete loan lifecycle and coordinating all system components.

**Key Features**:
- **Loan Creation**: Creates loans with proper collateral locking
- **Repayment Processing**: Handles partial and full loan repayments
- **Interest Calculation**: Computes time-based interest on outstanding loans
- **System Statistics**: Tracks total loans, debt, and system health

**Loan Structure**:
```move
struct Loan {
    loan_id: u64,
    borrower: address,
    collateral_amount: u64,
    loan_amount: u64,
    outstanding_balance: u64,
    interest_rate: u64,
    creation_timestamp: u64,
    state: u8, // 0=Active, 1=Repaid, 2=Defaulted
}
```

**Main Functions**:
- `create_loan()` - Create new loan with collateral locking (admin only)
- `repay_loan()` - Process loan repayments with interest calculation
- `close_loan()` - Emergency loan closure (admin only)
- `get_loan()` - Retrieve complete loan information
- `get_borrower_loans()` - List all loans for a borrower

**Interest Calculation**:
- **Formula**: `(principal × rate × time) / (10000 × seconds_per_year)`
- **Compounding**: Simple interest calculation
- **Payment Priority**: Interest paid first, then principal

---

### 6. 🧪 xBTC Token (`xbtc_token.move`)

**Purpose**: Mock Bitcoin token for testing and development purposes.

**Key Features**:
- **Unlimited Minting**: Admin can mint any amount for testing
- **No Real Value**: Purely for development and testing
- **Standard ERC20**: Full token functionality for realistic testing

**Main Functions**:
- `mint()` - Create new xBTC tokens (admin only)
- `burn()` - Destroy xBTC tokens (admin only)
- `transfer()` - Standard token transfers
- `mint_to_self()` - Convenience function for testing

**Token Details**:
- **Name**: "Mock Bitcoin for Testing"
- **Symbol**: "xBTC"
- **Decimals**: 8 (same as Bitcoin)

---

## 🔄 System Workflow

### Depositing Collateral
1. User calls `collateral_vault::deposit_collateral()`
2. Vault updates user's collateral balance
3. Vault mints ctrlBTC tokens to user
4. User receives ctrlBTC as proof of deposit

### Creating a Loan
1. Admin calls `loan_manager::create_loan()` with borrower details
2. LoanManager gets interest rate from InterestRateModel
3. LoanManager locks collateral in CollateralVault
4. LoanManager mints lnBTC tokens to borrower
5. Loan record is created and tracked

### Repaying a Loan
1. Admin calls `loan_manager::repay_loan()` with repayment amount
2. LoanManager calculates interest owed
3. Payment is split between interest and principal
4. If full repayment, collateral is unlocked
5. Loan state is updated accordingly

### Withdrawing Collateral
1. User calls `collateral_vault::withdraw_collateral()`
2. Vault checks available (unlocked) collateral
3. User's ctrlBTC tokens are burned
4. Collateral balance is updated

---

## 🔐 Security Features

### Access Control
- **Admin Functions**: Protected by admin-only modifiers
- **Contract Integration**: Only authorized contracts can call restricted functions
- **Emergency Controls**: Pause functionality for emergency situations

### Safety Mechanisms
- **Reentrancy Protection**: Prevents recursive calls
- **Amount Validation**: Ensures positive amounts for all operations
- **State Verification**: Validates loan states before operations
- **Collateral Checks**: Ensures sufficient collateral before operations

### Error Handling
- **Comprehensive Error Codes**: Clear error messages for all failure cases
- **Input Validation**: Thorough validation of all function parameters
- **State Consistency**: Maintains consistent state across all operations

---

## 📈 System Statistics

The system tracks various metrics:
- **Total Active Loans**: Number of currently active loans
- **Total Outstanding Debt**: Sum of all principal amounts owed
- **Total Vault Collateral**: Amount of collateral held in the vault
- **Individual Loan Details**: Complete loan information and history

---

## 🚀 Deployment Notes

1. **Initialization Order**:
   - Deploy and initialize InterestRateModel
   - Deploy and initialize ctrlBTC and lnBTC tokens
   - Deploy and initialize CollateralVault
   - Deploy and initialize LoanManager with contract addresses

2. **Configuration**:
   - Set appropriate interest rates in InterestRateModel
   - Configure admin addresses for all contracts
   - Set up proper access controls between contracts

3. **Testing**:
   - Use xBTC token for development testing
   - Verify all contract integrations work correctly
   - Test emergency pause/unpause functionality

---

## 📚 Additional Resources

- **Move Language Documentation**: [https://move-language.github.io/move/](https://move-language.github.io/move/)
- **Aptos Developer Docs**: [https://aptos.dev/](https://aptos.dev/)
- **Contract ABIs**: Available in `contract_abis.json`
- **Deployment Scripts**: Available in `scripts/` directory
