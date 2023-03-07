# Working with Authorization Keys

This tutorial demonstrates how to retrieve and use the authorization keys associated with a Deploy, using the `list_authorization_keys` function. <!-- TODO add link to docs.rs when 1.5 ships. --> 

```rust
let authorization_keys = runtime::list_authorization_keys();
```

Remember that authorization keys are listed in a Deploy's [approvals](../../concepts/serialization-standard.md#serialization-standard-deploy), a struct containing the signatures and the public keys of the signers, which are also called authorizing keys.

```json
"approvals" : [
	0 : {

		"signer":"0140a48b549ae33cf28e39241a33dd5e22f491d8811f9d83981f3549d418e06da0"
		"signature":"0146f3e070701328e2fbe4347a0a3a82b25f72cdaf8808145a60af5dadc6601efd02806d70a4da24db7a2f74360a6331171a6023f3c81a47d122403d87b409e80a"
	}
]
```

## Prerequisites

- You meet the [development prerequisites](../../developers/prerequisites.md) and are familiar with [writing and testing on-chain code](/writing-contracts/)
- You know how to [send and verify deploys](../../developers/dapps/sending-deploys.md)
- You are familiar with these concepts:
   - [Casper Accounts](https://docs.casperlabs.io/concepts/serialization-standard/#serialization-standard-account) 
   - [Deploys](https://docs.casperlabs.io/concepts/serialization-standard/#serialization-standard-deploy)
   - [Associated Keys](https://docs.casperlabs.io/concepts/serialization-standard/#associatedkey)
   - [Approvals](https://docs.casperlabs.io/concepts/serialization-standard/#approval), also known as authorization keys

## Workflow

To start, clone the [authorization-keys-example](https://github.com/casper-ecosystem/authorization-keys-example/) repository from GitHub, prepare your Rust environment and build the tests with the following commands.

```bash
git clone ...
cd ...
make prepare
make test
```

Upon installation, the contract code stores the authorization keys into a NamedKey. The contract also contains an entry-point that returns the intersection of the caller's authorization keys and the authorization keys saved during contract installation.

The example repository contains 3 folders:
- `client` - session code that interacts with the blockchain containing two Wasm files:
   - `add_keys.wasm` - session code that adds an associated key to the calling account
   - `contract_call.wasm` - session code that checks if one of the entry-point caller's authorization keys is listed in the contract's installer authorization keys
- `contract` - a simple contract that demonstrates the usage of authorization keys and compiles into a `contract.wasm` file
- `tests` - tests and supporting utilities to verify and demonstrate the contract's expected behavior

### Testing the authorization keys

This section highlights the tests written for this example, to verify the usage of authorization keys. The tests are divided into 3 sections:
* Testing the contract installation
* Testing the contract's unique entry point
* Testing the entry point using a client contract call


#### `should_allow_install_contract_with_default_account`

The first test demonstrates contract installation within the context of the caller's account. In this test, the caller's account hash is in the list of authorization keys associated with the Deploy.
<!-- TODO add link to Github integration_tests.rs#L28 -->

```
	let deploy_item = DeployItemBuilder::new()
            .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
            .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR])
            .with_address(*DEFAULT_ACCOUNT_ADDR)
            .with_session_code(session_code, session_args)
            .build();

```

#### `should_disallow_install_with_non_added_authorization_key`

The second test tries to sign the installer Deploy with an authorization key that is not part of the caller's associated keys. This is not allowed, because the authorization keys used to sign a Deploy need to be a subset of the caller's associated keys. So, the installer Deploy fails as expected.
<!-- TODO add link to Github integration_tests.rs#L57 -->

```
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

The third test demonstrates a successful installer Deploy using an added authorization key. <!-- TODO add link to Github integration_tests.rs#L83 -->

The test code calls session code to add an associated account (`ACCOUNT_USER_1`) to the caller's associated keys.

```
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

Now that the associated keys contain the ACCOUNT_USER_1, the deploy can be signed by ACCOUNT_USER_1. Thus, you will see this account in the list of authorization keys for the installer Deploy. <!-- Line 191 -->


```
	let deploy_item = DeployItemBuilder::new()
            .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
            .with_authorization_keys(&[*DEFAULT_ACCOUNT_ADDR, account_addr_1])
            .with_address(*DEFAULT_ACCOUNT_ADDR)
            .with_session_code(session_code, session_args)
            .build();

        let execute_request = ExecuteRequestBuilder::from_deploy_item(deploy_item).build();
        builder.exec(execute_request).commit().expect_success();
```

#### `should_allow_entry_point_with_installer_authorization_key`

#### `should_allow_entry_point_with_account_authorization_key`

This test is the main test in this repository. 

#### `should_disallow_entry_point_without_authorization_key`

#### `should_allow_entry_point_through_contract_call_with_authorization_key`

This test validates the entry point using a client contract call, where the immediate caller is the contract, the caller is the account, and the authorization_keys are "forwarded".


#### `should_disallow_entry_point_through_contract_call_without_authorization_key`
