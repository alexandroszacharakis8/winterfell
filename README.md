# Winterfell 🐺

<a href="https://github.com/novifinancial/winterfell/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
<img src="https://github.com/novifinancial/winterfell/workflows/CI/badge.svg?branch=main">
<a href="https://deps.rs/repo/github/novifinancial/winterfell"><img src="https://deps.rs/repo/github/novifinancial/winterfell/status.svg"></a>
<img src="https://img.shields.io/badge/prover-rustc_1.51+-lightgray.svg">
<img src="https://img.shields.io/badge/verifier-rustc_1.51+-lightgray.svg">

An experimental project for building a distributed STARK prover.

**WARNING:** This is a research project. It has not been audited and may contain bugs and security flaws. This implementation is NOT ready for production use.

## Overview

A STARK is a novel proof-of-computation scheme to create efficiently verifiable proofs of the correct execution of a computation. The scheme was developed by Eli Ben-Sasson, Michael Riabzev at al. at Technion - Israel Institute of Technology. STARKs do not require an initial trusted setup, and rely on very few cryptographic assumptions. See [references](#References) for more info.

The aim of this project is to build a feature-rich, easy to use, and highly performant STARK prover which can generate integrity proofs for very large computations. STARK proof generation process is massively parallelizable, however, it also requires lots of RAM. For very large computations, amount of RAM available on a single machine may not be sufficient to efficiently generate a proof. Therefore, our final goal is to efficiently distribute proof generation across many machines.

### Status and features

While our distributed STARK prover implementation is still under development, this project contains fully-functional, multi-threaded, STARK prover and verifier code with the following nice properties:

**A simple interface.** This library provides a relatively simple interface for describing general computations. See [usage](#Usage) for a quick tutorial, [common crate](common) for the description of the interface, and [examples crate](examples) for a few real-world examples.

**Multi-threaded proof generation.** When compiled with `concurrent` feature enabled, the proof generation process will run in multiple threads. The library also supports concurrent construction of execution trace tables. The [performance](#Performance) section showcases the benefits of multi-threading.

**Configurable fields.** Both the base and the extension field for proof generation can be chosen dynamically. This simplifies fine-tuning of proof generation for specific performance and security targets. See [math crate](math) for description of currently available fields.

**Configurable hash functions.** The library allows dynamic selection of hash functions used in the STARK protocol. Currently, BLAKE3 and SHA3 hash functions are supported, and support for arithmetization-friendly hash function (e.g. Rescue) is planned.

#### Planned features

Over time, we hope extend the library with additional features:

**Distributed prover.** Distributed proof generation is the main priority of this project, and we hope to release an update containing it soon.

**Perfect zero-knowledge.** The current implementation provides succinct proofs but NOT perfect zero-knowledge. This means that, in its current form, the library may not be suitable for use cases where proofs must not leak any info about secret inputs. 

**WebAssembly support.** The library is written in pure Rust, and in the future we hope to enable compilation to WebAssembly. First, for the verifier, and later for the prover.

### Project structure
The project is organized into several crates like so:

| Crate                | Description |
| -------------------- | ----------- |
| [examples](examples) | Contains examples of generating/verifying proofs for several toy and real-world computations. |
| [prover](prover)     | Contains implementations of STARK provers. You can use these to generate computational integrity proofs. |
| [verifier](verifier) | Contains an implementation of a STARK verifier which can verify proofs generated by a prover. |
| [fri](fri)           | Contains implementation of a FRI prover and verifier. These are used internally by the STARK prover and verifier. |
| [common](common)     | Contains modules with components shared between the prover and the verifier. |
| [math](math)         | Contains modules with math operations needed in STARK proof generation/verification. These include: finite field arithmetic, polynomial arithmetic, and FFTs. |
| [crypto](crypto)     | Contains modules with cryptographic operations needed in STARK proof generation/verification. Specifically: hash functions and Merkle trees. |
| [utils](utils)       | Contains a few utility functions used throughout the library. |

## Usage
Generating STARK proofs for a computation is a relatively complicated process. This library aims to abstract away most of the complexity, however, the users are still expected to provide descriptions of their computations in a STARK-specific format. This format is called *algebraic intermediate representation*, or AIR, for short.

This library contains several higher-level constructs which make defining AIRs for computations a little easier, and there are also examples of AIRs for several computations available in the [examples crate](examples). However, the best way to understand the STARK proof generation process is to go through a trivial example from start to finish.

First, we'll need to pick a computation for which we'll be generating and verifying STARK proofs. To keep things simple, we'll use the following:

```Rust
use math::field::{f128::BaseElement, FieldElement};

fn do_work(start: BaseElement, n: usize) -> BaseElement {
    let mut result = start;
    for _ in 1..n {
        result = result.exp(3) + BaseElement::new(42);
    }
    result
}
```

This computation starts with an element in a finite field and then, for the specified number of steps, cubes the element and adds value `42` to it.

Suppose, we run this computation for a million steps and get some result. Using STARKs we can prove that we did the work correctly without requiring any verifying party to re-execute the computation. Here is how to do it.

First, we need to define an *execution trace* for our computation. This trace should capture the state of the computation at every step of its execution. In our case, the trace is just a single column of intermediate values after each execution of the loop. For example, if we start with value `3` and run the computation for 1,048,576 (same as 2<sup>20</sup>) steps, the execution trace will look like this:

| Step      | State  |
| :-------: | :----- |
| 0         | 3      |
| 1         | 69     |
| 2         | 328551 |
| 3         | 35465687262668193 |
| 4         | 237280320818395402166933071684267763523 |
| ...       |
| 1,048,575 | 247770943907079986105389697876176586605 |

To record the trace, we'll use the `ExecutionTrace` struct provided by the library. The function below, is just a modified version of the `do_work()` function which records every intermediate state of the computation in the `ExecutionTrace` struct:

```Rust
use math::field::{f128::BaseElement, FieldElement};
use prover::ExecutionTrace;

pub fn build_do_work_trace(start: BaseElement, n: usize) -> ExecutionTrace<BaseElement> {
    // Instantiate the trace with a given width and length; this will allocate all
    // required memory for the trace
    let trace_width = 1;
    let mut trace = ExecutionTrace::new(trace_width, n);

    // Fill the trace with data; the first closure initializes the first state of the
    // computation; the second closure computes the next state of the computation based
    // on its current state.
    trace.fill(
        |state| {
            state[0] = start;
        },
        |_, state| {
            state[0] = state[0].exp(3u32.into()) + BaseElement::new(42);
        },
    );

    trace
}
```

Next, we need to define the AIR for our computation. We do this by implementing the `Air` trait. At the high level, the code below does three things:

1. Defines what the public inputs for our computation should look like. These inputs are called "public" because they must be known to both, the prover and the verifier.
2. Defines a transition function with a single transition constraint. This transition constraint must evaluate to zero for all valid state transitions, and to non-zero for any invalid state transition. The degree of this constraint is 3 (see more about constraint degrees [here](common/#Constraint-degrees)).
3. Define two assertions against an execution trace of our computation. These assertions tie a specific set of public inputs to a specific execution trace (see more about assertions [here](common/#Trace-assertions)).

For more information about the `Air` trait see [common crate](common#Air-trait), and here is the actual code:

```Rust
use math::field::{f128::BaseElement, FieldElement};
use prover::{
    Air, Assertion, ComputationContext, EvaluationFrame, ProofOptions, Serializable,
    TraceInfo, TransitionConstraintDegree,
};

// Public inputs for our computation will consist of the starting value and the end result.
pub struct PublicInputs {
    start: BaseElement,
    result: BaseElement,
}

// We need to describe how public inputs can be converted to bytes.
impl Serializable for PublicInputs {
    fn write_into(&self, target: &mut Vec<u8>) {
        self.start.write_into(target);
        self.result.write_into(target);
    }
}

// For a specific instance of our computation, we'll keep track of the public inputs and
// the computation's context which we'll build in the constructor. The context is used
// internally by the Winterfell prover/verifier when interpreting this AIR.
pub struct WorkAir {
    context: ComputationContext,
    start: BaseElement,
    result: BaseElement,
}

impl Air for WorkAir {
    // First, we'll specify which finite field to use for our computation, and also how
    // the public inputs must look like.
    type BaseElement = BaseElement;
    type PublicInputs = PublicInputs;

    // Here, we'll construct a new instance of our computation which is defined by 3 parameters:
    // starting value, number of steps, and the end result. Another way to think about it is that
    // an instance of our computation is a specific invocation of the do_work() function.
    fn new(trace_info: TraceInfo, pub_inputs: PublicInputs, options: ProofOptions) -> Self {
        // since our execution trace has only one column, we set trace width to 1; trace length
        // depends on the a specific execution instance of the computation, and so is passed in
        // as a construction parameter.
        let trace_width = 1;
        let trace_length = trace_info.length;

        // Our computation requires a single transition constraint. The constraint itself 
        // is defined in the evaluate_transition() method below, but here we need to specify
        // the expected degree of the constraint. If the expected and actual degrees of the
        // constraints don't match, an error will be thrown in the debug mode, but in release
        // mode, an invalid proof will be generated which will not be accepted by any verifier.
        let degrees = vec![TransitionConstraintDegree::new(3)];
        WorkAir {
            context: ComputationContext::new(trace_width, trace_length, degrees, options),
            start: pub_inputs.start,
            result: pub_inputs.result,
        }
    }

    // In this method we'll define our transition constraints; a computation is considered to
    // be valid, if for all valid state transitions, transition constraints evaluate to all
    // zeros, and for any invalid transition, at least one constraint evaluates to a non-zero
    // value. The `frame` parameter will contain current and next states of the computation.
    fn evaluate_transition<E: FieldElement + From<Self::BaseElement>>(
        &self,
        frame: &EvaluationFrame<E>,
        _periodic_values: &[E],
        result: &mut [E],
    ) {
        // First, we'll read the current state, and use it to compute the expected next state
        let current_state = &frame.current[0];
        let next_state = current_state.exp(3u32.into()) + E::from(42u32);

        // Then, we'll subtract the expected next state from the actual next state; this will
        // evaluate to zero if and only if the expected and actual states are the same.
        result[0] = frame.next[0] - next_state;
    }

    // Here, we'll define a set of assertions about the execution trace which must be satisfied
    // for the computation to be valid. Essentially, this ties computation's execution trace
    // to the public inputs.
    fn get_assertions(&self) -> Vec<Assertion<Self::BaseElement>> {
        // for our computation to be valid, value in column 0 at step 0 must be equal to the 
        // starting value, and at the last step it must be equal to the result.
        let last_step = self.trace_length() - 1;
        vec![
            Assertion::single(0, 0, self.start),
            Assertion::single(0, last_step, self.result),
        ]
    }

    // This is just boilerplate which is used by the Winterfell prover/verifier to retrieve
    // the context of the computation.
    fn context(&self) -> &ComputationContext {
        &self.context
    }
}
```

Now, we are finally ready to generate a STARK proof. The function below, will execute our computation, and will return the result together with the proof that the computation was executed correctly.

```Rust
use math::field::{f128::BaseElement, FieldElement};
use prover::{self, ExecutionTrace, FieldExtension, HashFunction, ProofOptions, StarkProof};

pub fn prove_work() -> (BaseElement, StarkProof) {
    // We'll just hard-code the parameters here for this example.
    let start = BaseElement::new(3);
    let n = 1_048_576;

    // Build the execution trace and get the result from the last step.
    let trace = build_do_work_trace(start, n);
    let result = trace.get(0, n - 1);

    // Define proof options; these will be enough for ~96-bit security level.
    let options = ProofOptions::new(
        32, // number of queries
        32, // blowup factor
        0,  // grinding factor
        HashFunction::Blake3_256,
        FieldExtension::None,
    );

    // Generate the proof.
    let pub_inputs = PublicInputs { start, result };
    let proof = prover::prove::<WorkAir>(trace, pub_inputs, options).unwrap();

    (result, proof)
}
```

We can then give this proof (together with the public inputs) to anyone, and they can verify that we did in fact execute the computation and got the claimed result. They can do this like so:

```Rust
pub fn verify_work(start: BaseElement, result: BaseElement, proof: StarkProof) {
    // The number of steps and options are encoded in the proof itself, so we
    // don't need to pass them explicitly to the verifier.
    let pub_inputs = PublicInputs { start, result };
    match verifier::verify::<WorkAir>(proof, pub_inputs) {
        Ok(_) => println!("yay! all good!"),
        Err(_) => panic!("something went terribly wrong!"),
    }
}
```

That's all there is to it! As mentioned above, the [examples](examples) crate contains examples of much more interesting computations (together with instructions on how to compile and run these examples). So, do check it out.

## Performance
The Winterfell prover's performance depends on a large number of factors including the nature of the computation itself, efficiency of encoding the computation in AIR, proof generation parameters, hardware setup etc. Thus, the benchmarks below should be viewed as directional: they illustrate the general trends, but concrete numbers will be different for different computations, choices of parameters etc.

The computation we benchmark here is a chain of Rescue hash invocations (see [examples](examples/#Rescue-hash-chain) for more info). The benchmarks were run on Intel Core i9-9980KH @ 2.4 GHz and 32 GB of RAM using all 8 cores.

<table style="text-align:center">
    <thead>
        <tr>
            <th rowspan=2 style="text-align:left">Chain length</th>
            <th rowspan=2 style="text-align:center">Trace time</th>
            <th colspan=2 style="text-align:center">100 bit security</th>
            <th colspan=2 style="text-align:center">128 bit security</th>
            <th rowspan=2 style="text-align:center">R1CS equiv.</th>
        </tr>
        <tr>
            <th>Proving time</th>
            <th>Proof size</th>
            <th>Proving time</th>
            <th>Proof size</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="text-align:left">2<sup>10</sup></td>
            <td>0.1 sec</td>
            <td>0.04 sec</td>
            <td>71 KB</td>
            <td>0.07 sec</td>
            <td>108 KB</td>
            <td>2<sup>17</sup> constr.</td>
        </tr>
        <tr>
            <td style="text-align:left">2<sup>12</sup></td>
            <td>0.4 sec</td>
            <td>0.14 sec</td>
            <td>90 KB</td>
            <td>0.25 sec</td>
            <td>135 KB</td>
            <td>2<sup>19</sup> constr.</td>
        </tr>
        <tr>
            <td style="text-align:left">2<sup>14</sup></td>
            <td>1.4 sec</td>
            <td>0.6 sec</td>
            <td>114 KB</td>
            <td>1 sec</td>
            <td>170 KB</td>
            <td>2<sup>21</sup> constr.</td>
        </tr>
        <tr>
            <td style="text-align:left">2<sup>16</sup></td>
            <td>6 sec</td>
            <td>2.5 sec</td>
            <td>142 KB</td>
            <td>4 sec</td>
            <td>210 KB</td>
            <td>2<sup>23</sup> constr.</td>
        </tr>
        <tr>
            <td style="text-align:left">2<sup>18</sup></td>
            <td>24 sec</td>
            <td>11 sec</td>
            <td>168 KB</td>
            <td> 18 sec </td>
            <td> 245 KB </td>
            <td>2<sup>25</sup> constr.</td>
        </tr>
        <tr>
            <td style="text-align:left">2<sup>20</sup></td>
            <td>94 sec</td>
            <td>50 sec</td>
            <td>199 KB</td>
            <td> 104 sec </td>
            <td> 290 KB </td>
            <td>2<sup>27</sup> constr.</td>
        </tr>
    </tbody>
</table>

A few remarks about these benchmarks:
* **Trace time** is the time it takes to generate an execution trace for the computation. This time does not depend on the chosen security level. For this specific computation, trace generation must be sequential, and thus, cannot take advantage of multiple cores. However, for other computations, where execution trace can be generated in parallel, trace time would be much smaller in relation to the proving time.
* **R1CS equiv.** is a very rough estimate of how many R1CS constraints would be required for this computation. The assumption here is that a single invocation of Rescue hash function requires ~120 R1CS constraints.
* As can be seen from the table, with STARKs, we can dynamically trade off proof size, proof security level, and proving time against each other.

As mentioned previously, we used all 8 CPU cores for the above benchmark. In general, Winterfell prover performance scales nearly linearly with every additional CPU core. This is because nearly all steps of STARK proof generation process can be parallelized. The table below illustrates this relationship on the example of 2<sup>16</sup> hash invocations.

| Threads | Trace time  | Proving time (100-bit security) | Proving time (128-bit security) |
| ------- | :---------: | :--------------------: | :--------------------: |
| 1       | 6 sec       | 16.4 sec               | 26.9 sec               |
| 2       | 6 sec       | 9 sec                  | 14 sec                 |
| 4       | 6 sec       | 4.8 sec                | 8 sec                  |
| 8       | 6 sec       | 3 sec                  | 4.8 sec                |
| 16      | 6 sec       | 2.6 sec                | 4.2 sec                |

A few remarks about these benchmarks:
* We are still using the same 8-core machine - thus, going from 8 to 16 threads has only a minor impact.
* Utilizing all 8 cores to the fullest, reduces prover time by 5x - 6x as compared to the single-threaded proof generation.

## References
If you are interested in learning how STARKs work under the hood, here are a few links to get you started. From the standpoint of this library, *arithmetization* is by far the most important concept to understand.

* STARKs whitepaper: [Scalable, transparent, and post-quantum secure computational integrity](https://eprint.iacr.org/2018/046)
* STARKs vs. SNARKs: [A Cambrian Explosion of Crypto Proofs](https://nakamoto.com/cambrian-explosion-of-crypto-proofs/)

Vitalik Buterin's blog series on zk-STARKs:
* [STARKs, part 1: Proofs with Polynomials](https://vitalik.ca/general/2017/11/09/starks_part_1.html)
* [STARKs, part 2: Thank Goodness it's FRI-day](https://vitalik.ca/general/2017/11/22/starks_part_2.html)
* [STARKs, part 3: Into the Weeds](https://vitalik.ca/general/2018/07/21/starks_part_3.html)

StarkWare's STARK Math blog series:
* [STARK Math: The Journey Begins](https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71)
* [Arithmetization I](https://medium.com/starkware/arithmetization-i-15c046390862)
* [Arithmetization II](https://medium.com/starkware/arithmetization-ii-403c3b3f4355)
* [Low Degree Testing](https://medium.com/starkware/low-degree-testing-f7614f5172db)
* [A Framework for Efficient STARKs](https://medium.com/starkware/a-framework-for-efficient-starks-19608ba06fbe)

License
-------

This project is [MIT licensed](./LICENSE).

Acknowledgements
-------

The original codebase was developed by Irakliy Khaburzaniya ([@irakliyk](https://github.com/irakliyk)) with contributions from François Garillot ([@huitseeker](https://github.com/huitseeker)), Kevin Lewi ([@kevinlewi](https://github.com/kevinlewi)), Konstantinos Chalkias ([@kchalkias](https://github.com/kchalkias)), and Jasleen Malvai ([@Jasleen1](https://github.com/Jasleen1)).
