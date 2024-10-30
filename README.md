# `sp1-solana`

This crate verifies Groth16 proofs generated with SP1, leveraging Solana's BN254 precompiles for efficient cryptographic operations.

> [!CAUTION]
>
> This repository is not audited for production use.

## Repository Overview

The `sp1-solana` library itself is in the [`verifier`](verifier) directory. [`example/program`](example/program) contains an example Solana program that uses this library to verify SP1 proofs, and [`example/script`](example/script) contains an example Solana script that invokes this program.

## Features

- **Groth16 Proof Verification**: Implements the Groth16 protocol for zero-knowledge proof verification.
- **Solana BN254 Precompiles**: Leverages Solana's native BN254 precompiles for optimized performance.
- **Easy Integration**: Seamlessly integrates with existing Solana programs and infrastructure.
- **Extensible**: Built with modularity in mind, allowing for future enhancements and integrations.

## Requirements

- Rust
- Edge Solana CLI
  - Install with the following command:

    ```shell
    sh -c "$(curl -sSfL https://release.anza.xyz/edge/install)"
    ```

## Example usage

The [example](./example) demonstrates how to use the `sp1-solana` crate to verify a proof generated by SP1 on Solana.
Running the script will perform the following steps:

1. Load an SP1 [`SP1ProofWithPublicValues`](https://docs.rs/sp1-sdk/2.0.0/sp1_sdk/proof/struct.SP1ProofWithPublicValues.html)
from the pre-generated proof [`fibonacci_proof.bin`](../proofs/fibonacci_proof.bin). This is a SP1 Groth16 proof that
proves that the 20th fibonacci number is 6765. Optionally, this proof can be freshly generated from
the [`sp1-program`](../sp1-program).

2. Extract the proof and public inputs from the `SP1ProofWithPublicValues`.

- The `proof` is the Groth16 proof, serialized in [SP1's standard format](https://docs.rs/sp1-sdk/2.0.0/sp1_sdk/proof/struct.SP1ProofWithPublicValues.html#method.bytes)
- The `sp1_public_inputs` are the public inputs to the underlying sp1 program.

Here is a snippet from the [example script](./example/script/src/main.rs) that demonstrates this.

```rust

/// The instruction data for the program.
#[derive(BorshDeserialize, BorshSerialize)]
pub struct SP1Groth16Proof {
    pub proof: Vec<u8>,
    pub sp1_public_inputs: Vec<u8>,
}

...

// Load the proof from the file, and extract the proof, public inputs, and program vkey hash.
let sp1_proof_with_public_values = SP1ProofWithPublicValues::load(&proof_file).unwrap();

let groth16_proof = SP1Groth16Proof {
    proof: sp1_proof_with_public_values.bytes(),
    sp1_public_inputs: sp1_proof_with_public_values.public_values.to_vec(),
};

// Send the proof to the contract, and verify it on `solana-program-test`.
run_verify_instruction(groth16_proof).await;
```

3. Using the [`solana-program-test`](https://docs.rs/solana-program-test/latest/solana_program_test/) framework, send the `SP1Groth16Proof` to the
[`fibonacci-verifier-contract`](./example/program). This smart contract will verify the proof using the `sp1-solana`
crate against the fibonacci SP1 program vkey and print out the public inputs.

> [!NOTE]
> In this example, a Groth16 proof and public values are directly passed into the contract as transaction data.
> In real use cases, this may not be reasonable, since the upper limit for transaction data is 1232 bytes.
> Groth16 proofs themselves are already 260 bytes, and public inputs can potentially be very large.
> See [this article](https://solana.com/developers/courses/program-optimization/lookup-tables) for a discussion
> on how to handle this.

Here is a snippet that demonstrates how to perform the verification and read the public inputs on chain.

```rust
// Derived by running `vk.bytes32()` on the program's vkey.
const FIBONACCI_VKEY_HASH: &str =
    "0083e8e370d7f0d1c463337f76c9a60b62ad7cc54c89329107c92c1e62097872";

pub fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    // Deserialize the SP1Groth16Proof from the instruction data.
    let groth16_proof = SP1Groth16Proof::try_from_slice(instruction_data)
        .map_err(|_| ProgramError::InvalidInstructionData)?;

    // Get the SP1 Groth16 verification key from the `sp1-solana` crate.
    let vk = sp1_solana::GROTH16_VK_2_0_0_BYTES;

    // Verify the proof.
    verify_proof(
        &groth16_proof.proof,
        &groth16_proof.sp1_public_inputs,
        &FIBONACCI_VKEY_HASH,
        vk,
    )
    .map_err(|_| ProgramError::InvalidInstructionData)?;

    // Print out the public values.
    let mut reader = groth16_proof.sp1_public_inputs.as_slice();
    let n = u32::deserialize(&mut reader).unwrap();
    let a = u32::deserialize(&mut reader).unwrap();
    let b = u32::deserialize(&mut reader).unwrap();
    msg!("Public values: (n: {}, a: {}, b: {})", n, a, b);

    Ok(())
}
```

### Running the script

To load the pregenerated proof and verify it on `solana-program-test`, run the following commands.

```shell
cd script
RUST_LOG=info cargo run --release
```

To generate a fresh proof from the program in `sp1-program`, run the following commands.

```shell
cd script
RUST_LOG=info cargo run --release -- --prove
```

### Deploying the Example Solana Program to Devnet

Run the following commands to build and deploy the example solana program to devnet. These commands
assume you've already created a Solana keypair locally, and you have the edge solana CLI tools.
Request [devnet sol](https://faucet.solana.com/) as necessary.

```shell
cd example/program
cargo build-sbf --sbf-out-dir ./target
solana config set -ud
solana program deploy --program-id target/fibonacci_verifier_contract-keypair.json target/fibonacci_verifier_contract.so
```

## Installation

Add `sp1-solana` to your `Cargo.toml`:

```toml
[dependencies]
sp1-solana = { git = "https://github.com/succinctlabs/sp1-solana" }
```

## Acknowledgements

This crate uses the [`groth16-solana`](https://github.com/Lightprotocol/groth16-solana/) crate from Light Protocol Labs for the Groth16 proof verification, and the [`ark-bn254`](https://github.com/arkworks-rs/algebra) crate for the elliptic curve operations.
