# Phantom Zone: Onboarding Guide

Welcome to the Phantom Zone project! This guide aims to provide a structured learning path for new contributors to understand the codebase, the cryptographic concepts, and how to effectively contribute.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Learning Path](#learning-path)
3. [Core Concepts](#core-concepts)
4. [Codebase Structure](#codebase-structure)
5. [Development Workflow](#development-workflow)
6. [Example-Driven Learning](#example-driven-learning)
7. [Advanced Topics](#advanced-topics)
8. [Troubleshooting](#troubleshooting)

## Prerequisites

Before diving into Phantom Zone, ensure you have:

- **Rust Knowledge**: Familiarity with Rust programming language (traits, generics, ownership)
- **Cryptography Basics**: Understanding of basic cryptographic concepts
- **Development Environment**: Rust toolchain installed (rustc, cargo)

If you need to brush up on any of these:
- [Rust Book](https://doc.rust-lang.org/book/)
- [Cryptography Basics](https://www.crypto101.io/)

## Learning Path

We recommend the following path to learn Phantom Zone:

### Stage 1: Understand the Problem Space (1-2 days)
- Read the README.md to understand the project's purpose
- Read the [Fully Homomorphic Encryption (FHE) primer](#fhe-primer) below
- Read the [Multi-Party Computation (MPC) overview](#mpc-overview) below

### Stage 2: Explore Examples (2-3 days)
- Run and study the examples in the following order:
  1. `non_interactive_fheuint8.rs`: Basic operations with encrypted integers
  2. `interactive_fheuint8.rs`: Interactive multi-party protocol
  3. `div_by_zero.rs`: Error handling in FHE
  4. `if_and_else.rs`: Conditional operations
  5. `meeting_friends.rs`: Real-world application of FHE
  6. `bomberman.rs`: Complex game logic with FHE

### Stage 3: Understand the Core Modules (1 week)
- Study the core modules in this sequence:
  1. `backend/`: Arithmetic operations
  2. `lwe.rs`: Basic encryption primitives
  3. `rgsw/`: Ring-GSW implementation
  4. `ntt.rs`: Number Theoretic Transform
  5. `pbs.rs`: Programmable Bootstrapping
  6. `bool/`: Boolean operations
  7. `shortint/`: FheUint8 implementation
  8. `multi_party.rs`: Multi-party protocols

### Stage 4: Make Your First Contribution (1-2 weeks)
- Fix a small bug or add a test
- Implement a small enhancement
- Create a new example

## Core Concepts

<a name="fhe-primer"></a>
### Fully Homomorphic Encryption (FHE) Primer

FHE allows computation on encrypted data without decryption. Key concepts:

1. **Homomorphic Property**: Operations on ciphertexts correspond to operations on plaintexts
2. **Noise**: Each operation increases "noise" in the ciphertext
3. **Bootstrapping**: Technique to refresh ciphertexts and reduce noise
4. **Cryptographic Primitives**:
   - **LWE (Learning With Errors)**: Basic encryption scheme
   - **RLWE (Ring-LWE)**: Extension of LWE to polynomials for efficiency
   - **RGSW (Ring-GSW)**: Advanced scheme for complex operations

<a name="mpc-overview"></a>
### Multi-Party Computation (MPC) Overview

MPC enables multiple parties to jointly compute a function over their inputs while keeping those inputs private:

1. **Secret Sharing**: Dividing a secret into shares
2. **Key Generation**: Distributed key generation without revealing individual secrets
3. **Interactive vs. Non-Interactive Protocols**:
   - **Interactive**: Requires communication rounds between parties
   - **Non-Interactive**: "One-shot" protocol with minimal communication
4. **Threshold Decryption**: Requiring multiple parties to decrypt results

### Phantom Zone Implementation

Phantom Zone combines FHE and MPC to enable:

1. **Private Inputs**: Multiple clients encrypt their private data
2. **Server Computation**: A server performs computations on encrypted data
3. **Collaborative Decryption**: Clients jointly decrypt the results

<a name="math-foundations"></a>
## Mathematical Foundations

This section provides a rigorous exploration of the mathematical principles behind Phantom Zone, essential for developers who want to understand or extend the core functionality.

### Lattice-Based Cryptography Fundamentals

#### Rings and Polynomial Operations

Phantom Zone operates in polynomial rings of the form $R = \mathbb{Z}[X]/(X^N + 1)$ where:
- $N$ is a power of 2 (typically 1024, 2048, or 4096)
- Ring elements are polynomials with integer coefficients modulo $X^N + 1$
- Operations in the ring involve polynomial addition, multiplication, and reduction

For computational efficiency, we work in the quotient ring $R_q = R/qR = \mathbb{Z}_q[X]/(X^N + 1)$ where:
- $q$ is the ciphertext modulus (usually a prime number)
- Coefficients are reduced modulo $q$

#### Learning With Errors (LWE)

The LWE problem, introduced by Regev, forms the security foundation:

**Definition**: Given many samples $(a_i, b_i)$ where:
- $a_i \in \mathbb{Z}_q^n$ are uniformly random vectors
- $b_i = \langle a_i, s \rangle + e_i \mod q$ where $e_i$ is a small error term drawn from error distribution $\chi$
- The task is to find the secret vector $s \in \mathbb{Z}_q^n$

**LWE Encryption Scheme**:
- KeyGen: $s \leftarrow \mathbb{Z}_q^n$ (secret key)
- Encrypt($m \in \{0,1\}$): 
  1. Sample $a \leftarrow \mathbb{Z}_q^n$, $e \leftarrow \chi$
  2. Output ciphertext $(a, b = \langle a, s \rangle + e + \lfloor q/2 \rfloor \cdot m)$
- Decrypt$(a, b)$: $m = \lfloor (2/q) \cdot (b - \langle a, s \rangle) \rceil \mod 2$

#### Ring-LWE (RLWE)

RLWE transfers the LWE problem to polynomial rings:

**Definition**: Given samples $(a_i, b_i)$ where:
- $a_i \in R_q$ are uniformly random polynomials
- $b_i = a_i \cdot s + e_i$ where $e_i$ is a small error polynomial (coefficients from $\chi$)
- The task is to find the secret polynomial $s \in R_q$

**RLWE Encryption Scheme**:
- KeyGen: $s \leftarrow R_q$ (secret key)
- Encrypt($m \in R_q$): 
  1. Sample $a \leftarrow R_q$, $e \leftarrow \chi$ over $R$
  2. Output ciphertext $(a, b = a \cdot s + e + m)$
- Decrypt$(a, b)$: $m = b - a \cdot s$

### FHEW-Style Bootstrapping

Bootstrapping is critical for fully homomorphic encryption, refreshing noisy ciphertexts:

#### The Bootstrapping Process

1. Start with ciphertext $(a, b)$ encrypting $m$ under key $s$
2. Homomorphically evaluate the decryption function:
   - Create a function $f$ that computes $b - \langle a, s \rangle \mod q$
   - Encrypt the secret key $s$ under itself (recursive encryption)
   - Apply $f$ to obtain a fresh encryption of $m$

#### Blind Rotation

The core technique in FHEW-style bootstrapping:

1. Represent the decryption function as a lookup table:
   - Prepare a test vector $\mathbf{t}$ encoding the function output
   
2. Blind rotation operation:
   - For input ciphertext $(a, b)$, we perform:
   
   $\mathbf{t}' = X^{-b} \cdot \prod_{i=0}^{n-1} (X^{a_i \cdot q/2} \cdot \text{CMUX}_i + (1 - \text{CMUX}_i))^{s_i} \cdot \mathbf{t}$
   
   Where:
   - $\text{CMUX}_i$ is a homomorphic selector controlled by bit $s_i$
   - The product rotates the test vector according to $b - \langle a, s \rangle$
   
3. This operation uses evaluation keys of the form:
   - $\text{evk}_i = \text{RGSW}(\text{Encrypt}(s_i))$
   - These keys enable homomorphic multiplication by secret bits

4. Optimized Blind Rotation (from paper 1):
   - Reduces the number of automorphisms needed
   - Key equation: $X^{\langle a, s \rangle} = \prod_{i=0}^{n-1} X^{a_i \cdot s_i}$
   - Improves performance while supporting arbitrary key distributions

### Multi-Party Computation Protocols

Phantom Zone implements two MPC protocols:

#### Non-Interactive Multi-Party Protocol

1. **Distributed Key Generation**:
   - Each party $i$ generates key pair $(p_i \approx a \cdot s_i, s_i)$
   - No interaction required between parties
   
2. **Mathematical Foundation**:
   - Public key: $p = \sum_{i=1}^{k} p_i$
   - Implicit secret key: $s = \sum_{i=1}^{k} s_i$ (never reconstructed)
   - Encryption works with $p$ directly
   - This construction is provably secure under RLWE assumptions

3. **Server Key Generation**:
   - Each party $i$ with secret $s_i$ generates a server key share
   - Server aggregates shares into single evaluation key
   - Security relies on the hardness of RLWE

#### Interactive Multi-Party Protocol

This protocol follows the BFV scheme extension described in paper 2:

1. **Round 1 (Public Key Generation)**:
   - Each party $i$ generates $(a, p_i = a \cdot s_i + e_i)$
   - Parties share public key pieces
   - Collective public key: $p = \sum_{i=1}^{k} p_i$

2. **Round 2 (Server Key Generation)**:
   - Parties use collective public key to generate server key shares
   - Shares support homomorphic operations

3. **Threshold Decryption**:
   - For ciphertext $(a, b)$, each party $i$ computes:
     $d_i = a \cdot s_i$
   - Decryption: $m = b - \sum_{i=1}^{k} d_i$

### Number Theoretic Transform (NTT)

NTT is critical for efficient polynomial operations:

1. **Definition**:
   - For polynomial $a(X) = \sum_{i=0}^{N-1} a_i X^i$, its NTT is:
     $\text{NTT}[a](j) = \sum_{i=0}^{N-1} a_i \omega^{ij} \mod q$
   - Where $\omega$ is a primitive $N$-th root of unity in $\mathbb{Z}_q$

2. **Key Properties**:
   - Transforms polynomial multiplication into point-wise multiplication
   - $\text{NTT}[a \cdot b] = \text{NTT}[a] \odot \text{NTT}[b]$
   - Reduces complexity from $O(N^2)$ to $O(N \log N)$

3. **Implementation Details**:
   - Phantom Zone uses optimized negacyclic NTT for the ring $\mathbb{Z}_q[X]/(X^N + 1)$
   - Implements the Cooley-Tukey butterfly algorithm
   - Pre-computes twiddle factors for efficiency

## Codebase Structure

### Module Dependencies

```
                    +-----------+
                    |  lib.rs   |
                    +-----+-----+
                          |
          +---------------+---------------+
          |               |               |
+---------v----+  +-------v-------+  +----v--------+
|   backend/   |  |     bool/     |  |  shortint/  |
+------+-------+  +-------+-------+  +------+------+
       |                  |                 |
       |         +--------v--------+        |
       +---------> lwe.rs, pbs.rs <--------+
                 |    rgsw/,      |
                 |    ntt.rs      |
                 +--------+-------+
                          |
                 +--------v--------+
                 | multi_party.rs  |
                 +-----------------+
```

### Key Module Descriptions

#### Low-Level Cryptographic Primitives

1. **backend/**: Foundation for modular arithmetic
   - `modulus_u64.rs`: Modular arithmetic operations for u64
   - `power_of_2.rs`: Special case optimizations for power-of-2 moduli
   - `word_size.rs`: Word-size specific operations

2. **lwe.rs**: Learning With Errors implementation
   - Basic encryption/decryption
   - Core homomorphic addition

3. **rgsw/**: Ring-GSW (Gentry-Sahai-Waters) implementation
   - `keygen.rs`: Key generation for RGSW
   - `runtime.rs`: Runtime operations for RGSW
   - Advanced homomorphic operations

4. **ntt.rs**: Number Theoretic Transform
   - Fast polynomial operations in rings
   - Critical for efficient implementation of RLWE

5. **pbs.rs**: Programmable Bootstrapping
   - Noise management in ciphertexts
   - Function evaluation on encrypted data

#### Higher-Level Abstractions

6. **bool/**: Boolean operations on encrypted data
   - `evaluator.rs`: Evaluator for boolean circuits
   - `keys.rs`: Key management
   - `mp_api.rs`/`ni_mp_api.rs`: Multi-party protocols

7. **shortint/**: 8-bit unsigned integer operations
   - `enc_dec.rs`: Encryption/decryption
   - `ops.rs`: Arithmetic/comparison operations

8. **multi_party.rs**: Multi-party computation protocols
   - Key sharing
   - Decryption sharing

## Development Workflow

### Setting Up Development Environment

1. Clone the repository:
   ```bash
   git clone https://github.com/phantomzone-org/phantom-zone.git
   cd phantom-zone
   ```

2. Build the project:
   ```bash
   cargo build
   ```

3. Run tests:
   ```bash
   cargo test
   # With features:
   cargo test --features "non_interactive_mp"
   cargo test --features "interactive_mp"
   ```

### Common Development Tasks

#### Adding a New Operation

1. Define the trait in the appropriate module
2. Implement the low-level operation
3. Add high-level API to make it user-friendly
4. Write tests
5. Create an example

#### Performance Optimization

1. Profile using benchmarks: `cargo bench`
2. Focus optimization on:
   - NTT operations
   - Bootstrapping operations
   - Memory usage in matrix operations

#### Adding a New Integer Type

1. Start by copying `shortint/` module structure
2. Adjust bit width and parameters
3. Implement appropriate operations
4. Update tests and examples

## Example-Driven Learning

Each example in the `examples/` directory demonstrates specific aspects of Phantom Zone:

### 1. non_interactive_fheuint8.rs

**Concepts covered:**
- Setting up the non-interactive protocol
- Encrypting private inputs
- Server-side computation
- Multi-party decryption

**Key learning points:**
- Server key share generation
- Key switching for client inputs
- Function evaluation on encrypted data
- Decryption share generation and aggregation

### 2. interactive_fheuint8.rs

**Concepts covered:**
- Two-round interactive protocol
- Public key share generation and exchange
- Collective public key usage

**Key learning points:**
- Differences between interactive and non-interactive
- Exchange of public key shares
- Role of the collective public key

### 3. div_by_zero.rs

**Concepts covered:**
- Error handling in FHE
- Thread-local error flags

**Key learning points:**
- How division by zero is handled
- Accessing and resetting error flags

### 4. if_and_else.rs

**Concepts covered:**
- Conditional operations on encrypted data
- Multiplexing (mux) operations

**Key learning points:**
- Challenges of branching in encrypted domain
- Implementation of mux for conditional selection

### 5. meeting_friends.rs

**Concepts covered:**
- Practical application of FHE
- Private location matching

**Key learning points:**
- Building simple applications with FHE
- Privacy-preserving distance calculation

### 6. bomberman.rs

**Concepts covered:**
- Complex game logic with encrypted data
- Multiple client interactions

**Key learning points:**
- Advanced use cases for FHE
- Encrypted game state management

## Advanced Topics

### Parameter Selection

Parameter selection is critical for security and performance:

- **Lattice Dimension**: Higher is more secure but slower
- **Error Distribution**: Affects noise growth and security
- **Modulus**: Balances precision and security
- **Decomposition Parameters**: Affects bootstrapping performance

The project provides parameter sets tuned for different party counts:
- `LTE2Party`: For ≤ 2 parties
- `LTE4Party`: For ≤ 4 parties
- `LTE8Party`: For ≤ 8 parties

### Security Considerations

When working with Phantom Zone, keep in mind:

1. **Decryption Share Generation**: Generate only one share per ciphertext
2. **Application Seed Reuse**: Avoid running MPC with the same seed multiple times
3. **Side-Channel Attacks**: Consider timing and memory access patterns

### Performance Optimization

Key areas for optimization:

1. **NTT Operations**: Most compute-intensive part
2. **PBS Operations**: Critical for refreshing ciphertexts
3. **Memory Usage**: Matrix operations can consume significant memory
4. **Parallelism**: Many operations are parallelizable

## Troubleshooting

### Common Issues

1. **Compilation Errors with Features**:
   - Ensure you're using the right feature flags for your protocol
   - Interactive: `--features "interactive_mp"`
   - Non-interactive: `--features "non_interactive_mp"`

2. **Decryption Failures**:
   - Check parameter selection for party count
   - Verify all decryption shares are correctly aggregated
   - Ensure operations don't introduce too much noise

3. **Performance Issues**:
   - Use the benchmarks to identify bottlenecks
   - Focus optimization on NTT and PBS operations
   - Consider reducing parameter sizes if possible

### Getting Help

- Open an issue on GitHub
- Review existing issues for similar problems
- Consult the references in README.md

## Next Steps

After completing this onboarding guide:

1. Pick a small issue to fix or enhancement to implement
2. Contribute an additional example
3. Improve documentation or tests
4. Explore performance optimizations
5. Investigate new cryptographic techniques that could benefit the project

Welcome to the Phantom Zone community! Your journey into secure multi-party computation begins here.