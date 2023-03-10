# Working with Authorization Keys

This tutorial demonstrates how to retrieve and use the authorization keys associated with a deploy, using the `list_authorization_keys` function. <!-- TODO add link to docs.rs when 1.5 ships. --> 

```rust
let authorization_keys = runtime::list_authorization_keys();
```

<!-- TODO Remove https://docs.casperlabs.io if this is moved to the docs from all the links. 
../../concepts/serialization-standard.md#serialization-standard-deploy
../../developers/prerequisites.md
../../developers/dapps/sending-deploys.md
-->

Remember that authorization keys are listed under a Deploy's [approvals](https://docs.casperlabs.io/concepts/serialization-standard.md#serialization-standard-deploy) section, which lists the signatures and the public keys of the signers, which are also called authorizing keys. Here is an example:

```json
"approvals": [
    {
      "signer": "02021a4da3d6f32ea3ebd2519e1a37a1b811671085bf4f1cf2a36b931344a99b756a",
      "signature": "02df8cdf0bff3bd93e831d24563d5acbefa0ed13814550e910d03208d5fb3c11770dd3d918784ec84342e53666eacf59aeecbf4ce0cdd60e167c4a4b20e4b8f481"
    }
]
```

**Note**: This tutorial highlights certain lines of code, but for a full working version of the code, visit [GitHub](https://github.com/casper-ecosystem/authorization-keys-example).

## Prerequisites

- You meet the [development prerequisites](https://docs.casperlabs.io/developers/prerequisites.md) and are familiar with [writing and testing on-chain code](https://docs.casperlabs.io/developers/writing-contracts/)
- You know how to [send and verify deploys](https://docs.casperlabs.io/developers/dapps/sending-deploys.md)
- You are familiar with these concepts:
   - [Casper Accounts](https://docs.casperlabs.io/concepts/serialization-standard/#serialization-standard-account) 
   - [Deploys](https://docs.casperlabs.io/concepts/serialization-standard/#serialization-standard-deploy)
   - [Associated Keys](https://docs.casperlabs.io/concepts/serialization-standard/#associatedkey)
   - [Approvals](https://docs.casperlabs.io/concepts/serialization-standard/#approval), also known as authorization keys

## Workflow

To start, clone the [authorization-keys-example](https://github.com/casper-ecosystem/authorization-keys-example/) repository from GitHub, prepare your Rust environment and build the tests with the following commands.

```bash
git clone https://github.com/casper-ecosystem/authorization-keys-example/
cd authorization-keys-example
make prepare
make test
```

<!-- TODO add a link to each folder -->
The example repository contains 3 folders:
- `client` - This is a client folder containing two Wasm files:
   - `add_keys.wasm` - Session code that adds an associated key to the calling account
   - `contract_call.wasm` - Session code that calls the contract's entry point and stores the result into a named key
- `contract` - A simple contract that demonstrates the usage of authorization keys and compiles into a `contract.wasm` file
- `tests` - Tests and supporting utilities to verify and demonstrate the contract's expected behavior

### Client Wasm files

#### `add_keys.wasm` 

This file contains session code that adds an associated key to the calling account. For more details and a similar example, visit the [Two-Party Multi-Signature](https://docs.casperlabs.io/resources/tutorials/advanced/two-party-multi-sig/) tutorial.

#### `contract_call.wasm`

This session code calls the contract's entry point, which returns the intersection between two sets of keys:
- the authorization keys that signed the deploy that installed the contract (referred to here as the installer deploy)
- the authorization keys that signed the deploy calling the entry point (referred to here as the caller deploy)
The resulting list is stored under a named key of the account calling this session code.

```rust
let key_name: String = runtime::get_named_arg(ARG_KEY_NAME);
    let intersection =
        runtime::call_contract::<Vec<AccountHash>>(contract_hash, ENTRY_POINT, runtime_args! {});
    runtime::put_key(&key_name, storage::new_uref(intersection).into());
}
```

### The example contract

Upon installation, this example contract stores the authorization keys that signed the installer deploy into a NamedKey. <!-- TODO link installation to contract/src/main.rs#L75 -->

```rust
#[no_mangle]
pub extern "C" fn init() {
    if runtime::get_key(AUTHORIZATION_KEYS_INSTALLER).is_none() {
        let authorization_keys: Vec<AccountHash> =
            runtime::list_authorization_keys().iter().cloned().collect();

        let authorization_keys: Key = storage::new_uref(authorization_keys).into();
        runtime::put_key(AUTHORIZATION_KEYS_INSTALLER, authorization_keys);
    }
}
```

The contract contains an entry point that returns the intersection of the caller deploy's authorization keys and the installer deploy's authorization keys saved during contract installation. The following usage of `runtime::list_authorization_keys` retrieves the set of account hashes representing the keys signing the caller deploy. <!-- TODO link usage to contract/src/main.rs#L52 -->

```rust
let authorization_keys_caller: Vec<AccountHash> =
    runtime::list_authorization_keys().iter().cloned().collect();
```

### Testing the authorization keys

This section highlights the tests written for this example, demonstrating the usage of authorization keys. The tests are divided into these parts:
* Testing the contract installation
* Testing the contract's unique entry point
* Testing the entry point using a client contract call

The following tests focus on testing the contract installation.

#### `should_allow_install_contract_with_default_account`

The first test demonstrates contract installation within the context of the caller's account. Since the caller's account is the default, the caller's account hash is in the list of authorization keys associated with the deploy.
<!-- TODO add link to Github integration_tests.rs#L28 -->

```rust
let deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR])
    .with_address(*DEFAULT_ACCOUNT_ADDR)
    .with_session_code(session_code, session_args)
    .build();

```

#### `should_disallow_install_with_non_added_authorization_key`

This test tries to sign the installer deploy with an authorization key that is not part of the caller's associated keys. This is not allowed, because the authorization keys used to sign a deploy need to be a subset of the caller's associated keys. So, the installer deploy fails as expected.
<!-- TODO add link to Github integration_tests.rs#L57 -->

```rust
let deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR, account_addr_1])
    .with_address(*DEFAULT_ACCOUNT_ADDR)
    .with_session_code(session_code, session_args)
    .build();

let execute_request = ExecuteRequestBuilder::from_deploy_item(deploy_item).build();
builder.exec(execute_request).commit().expect_failure();
let error = builder.get_error().expect("must have error");
assert_eq!(error.to_string(), "Authorization failure: not authorized.");
```

#### `should_allow_install_with_added_authorization_key`

This test demonstrates a successful installer deploy using an added authorization key. After the initial test framework setup, the test calls session code to add an associated account (`account_addr_1`) to the default account's associated keys. <!-- TODO add link to Github integration_tests.rs#L83 -->

```rust
// Add ACCOUNT_USER_1 to DEFAULT_ACCOUNT_ADDR associated keys
let session_code = PathBuf::from(ADD_KEYS_WASM);
let session_args = runtime_args! {
    ASSOCIATED_ACCOUNT => account_addr_1
};

let add_keys_deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR])
    .with_address(*DEFAULT_ACCOUNT_ADDR)
    .with_session_code(session_code, session_args)
    .build();

let add_keys_execute_request =
    ExecuteRequestBuilder::from_deploy_item(add_keys_deploy_item).build();

builder
    .exec(add_keys_execute_request)
    .commit()
    .expect_success();
```

Since the default account's associated keys contain the newly added account `account_addr_1`, the default account can sign the deploy with this newly added key. Thus, you will see `account_addr_1` in the list of authorization keys for the installer deploy. <!-- integration_tests.rs#L191 -->

```rust
let deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR, account_addr_1])
    .with_address(*DEFAULT_ACCOUNT_ADDR)
    .with_session_code(session_code, session_args)
    .build();

let execute_request = ExecuteRequestBuilder::from_deploy_item(deploy_item).build();
builder.exec(execute_request).commit().expect_success();
```

The next tests exercise the contract's unique entry point to calculate the intersection between the caller deploy's authorization keys and the installer deploy's authorization keys.

#### `should_allow_entry_point_with_installer_authorization_key` 

<!-- TODO add link to Github integration_tests.rs#L144 -->
This test builds on the previous test which adds an associated account to the default account's associated keys and installs the contract using two authorization keys (the default account hash and the associated account hash `account_addr_1`). Additionally, the test invokes the contract's entry point using a deploy that is signed only by `account_addr_1` and runs under `account_addr_1`. This is possible because the deploy action threshold for `account_addr_1` is 1.

The additional logic invoking the entry point starts on line 201. <!-- TODO add link to Github integration_tests.rs#L201 -->

```rust
let entry_point_deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[account_addr_1])
    .with_address(account_addr_1)
    .with_stored_session_hash(contract_hash, ENTRYPOINT, runtime_args! {})
    .build();

let entry_point_request =
    ExecuteRequestBuilder::from_deploy_item(entry_point_deploy_item).build();

builder.exec(entry_point_request).expect_success().commit();
```

The entry point returns a resulting intersection of the caller deploy's authorization keys and installer deploy's authorization keys, which is a list containing the hash `account_addr_1`. Thus, the caller deploy is expected to succeed.

#### `should_allow_entry_point_with_account_authorization_key`

<!-- TODO add link to Github integration_tests.rs#L224 -->
This is the main test in this repository. After installing the contract using the default account, the test adds the default account address to account `account_addr_1` as an associated key. 

```rust
let session_code = PathBuf::from(ADD_KEYS_WASM);
let session_args = runtime_args! {
    ASSOCIATED_ACCOUNT => *DEFAULT_ACCOUNT_ADDR
};

let add_keys_deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[account_addr_1])
    .with_address(account_addr_1)
    .with_session_code(session_code, session_args)
    .build();
```

Then, the test creates a deploy to invoke the contract's entry point. This deploy is executed under `account_addr_1` and has two authorization keys, `account_addr_1` and the default account. Note that both authorization keys must sign the deploy to meet the deploy's new action threshold set to 2. The deploy should be executed successfully, because the resulting intersection should contain the default account hash. The default account used to sign the installer deploy is also in the authorization keys of the caller deploy.

```rust
let entry_point_deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_authorization_keys(&[account_addr_1, *DEFAULT_ACCOUNT_ADDR])
    .with_address(account_addr_1)
    .with_stored_session_hash(contract_hash, ENTRYPOINT, runtime_args! {})
    .build();

let entry_point_request =
    ExecuteRequestBuilder::from_deploy_item(entry_point_deploy_item).build();

builder.exec(entry_point_request).expect_success().commit();
```

#### `should_disallow_entry_point_without_authorization_key`

<!-- TODO add link to Github integration_tests.rs#L304 -->

The following tests exercise the entry point using a client contract call.

#### `should_allow_entry_point_through_contract_call_with_authorization_key`

<!-- TODO add link to Github integration_tests.rs#L403 -->

This test validates the entry point using a client contract call, where the caller is the account, the immediate caller is the contract, and the authorization_keys are "forwarded".

#### `should_disallow_entry_point_through_contract_call_without_authorization_key`

<!-- TODO add link to Github integration_tests.rs#L509 -->

