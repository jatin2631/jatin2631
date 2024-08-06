// src/wallet/main.rs

use ic_cdk::export::candid::{CandidType, Deserialize};
use ic_cdk_macros::{init, query, update};

#[derive(CandidType, Deserialize, Default)]
struct Wallet {
    owner: String,
    balance: u64,
}

static mut WALLET: Wallet = Wallet {
    owner: String::new(),
    balance: 0,
};

#[init]
fn init(owner: String) {
    unsafe {
        WALLET.owner = owner;
    }
    ic_cdk::println!("Wallet initialized for owner: {}", owner);
}

#[update]
fn send_tokens(amount: u64, recipient: String) -> Result<(), String> {
    ensure_owner()?;
    unsafe {
        if WALLET.balance < amount {
            return Err("Insufficient balance".to_string());
        }
        WALLET.balance -= amount;
        // Logic to send tokens to recipient (stubbed for simplicity)
        ic_cdk::println!("Sent {} tokens to {}", amount, recipient);
    }
    Ok(())
}

#[update]
fn receive_tokens(amount: u64) {
    unsafe {
        WALLET.balance += amount;
    }
    ic_cdk::println!("Received {} tokens", amount);
}

#[query]
fn get_balance() -> u64 {
    unsafe { WALLET.balance }
}

fn ensure_owner() -> Result<(), String> {
    let caller = ic_cdk::api::caller().to_string();
    unsafe {
        if WALLET.owner != caller {
            return Err("Only the owner can perform this action".to_string());
        }
    }
    Ok(())
}

// Unit tests
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_send_tokens() {
        unsafe {
            WALLET.owner = "owner".to_string();
            WALLET.balance = 100;
        }
        assert!(send_tokens(50, "recipient".to_string()).is_ok());
        assert_eq!(unsafe { WALLET.balance }, 50);
    }

    #[test]
    fn test_receive_tokens() {
        unsafe {
            WALLET.balance = 0;
        }
        receive_tokens(50);
        assert_eq!(unsafe { WALLET.balance }, 50);
    }

    #[test]
    fn test_get_balance() {
        unsafe {
            WALLET.balance = 100;
        }
        assert_eq!(get_balance(), 100);
    }

    #[test]
    fn test_insufficient_balance() {
        unsafe {
            WALLET.owner = "owner".to_string();
            WALLET.balance = 10;
        }
        assert_eq!(
            send_tokens(20, "recipient".to_string()),
            Err("Insufficient balance".to_string())
        );
    }

    #[test]
    fn test_only_owner_can_send() {
        unsafe {
            WALLET.owner = "owner".to_string();
            WALLET.balance = 100;
        }
        let non_owner = ic_cdk::export::Principal::anonymous();
        ic_cdk::api::set_caller(non_owner);
        assert_eq!(
            send_tokens(50, "recipient".to_string()),
            Err("Only the owner can perform this action".to_string())
        );
    }
}
