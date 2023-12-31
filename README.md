# badlock
Static reentrant deadlock detection for Rust programs using LLVM compiler infrastructure and taint analysis. 

We recommend you reopen this repo in VsCode with the [Remote Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension. This will automatically build the docker container and install all the necessary dependencies. Use the `rust-llvm` container configuration.

## Work In Progress (WIP)

🚧 **This project is currently a work in progress.** 🚧

## Overview
This project contains the following components:
- A generalized deadlock analysis crate, `badlock/lock-detection` using rust [`crepe`](https://github.com/ekzhang/crepe) (a tool for embedding datalog based programs in Rust).
- A Rust [`inkwell`](https://github.com/TheDan64/inkwell) and [`llvm-plugin`](https://github.com/jamesmth/llvm-plugin-rs) based LLVM pass, `badlock/llvm-lock-detection`, that can detect `std::sync::Mutex` reentrant deadlocks in Rust programs.
- A few e2e examples in `tests`.

### `badlock/lock-detection`
The `badlock/lock-detection` crate contains a generalized deadlock analysis that can be used to detect deadlocks in any program that can be modeled as a graph. The best entrypoint for understanding the crate is [here](./badlock/lock-detection//src//reentrant_lock_detection//facts.rs).

You can run several tests against mini-programs using `cargo test` in the `badlock/lock-detection` directory or by using the Rust Analyzer test running tools in VsCode. We highly recommend you check these tests in detail to understand how deadlocks are being detected.

We have additionally an abstraction layer over the `crepe` crate that allows for the use of generic types with `crepe` programs. This is used in the `badlock/llvm-lock-detection` crate to allow for the use of `inkwell` types with `crepe` programs.

### `badlock/llvm-lock-detection`
This crate contains an LLVM pass that can detect reentrant deadlocks in Rust programs. The pass is written using the `inkwell` and `llvm-plugin` crates. The pass is written in Rust and can be compiled to a shared library. You can later use this shared lib with `opt` to statically analyze LLVM IR compiled from a Rust program.

**WARNING**: owing to limitations in `inkwell` which are discussed below, the current implementation is quite hacky. This pass should be considered a proof of concept.

### `test`
To run the deadlock detection on all targets.
```
make
```

Inspect the analysis output in the `%.analysis` artifacts. You should see something like this...
```
MAY DEADLOCK!
__________
DEADLOCK #0
	FIRST LOCK: Symbol("InstructionValue { instruction_value: Value { name: \"\", address: 0xaaaacc70a6a0, is_const: false, is_null: false, is_undef: false, llvm_value: \"  call void @\\\"_ZN3std4sync5mutex14Mutex$LT$T$GT$4lock17h8dad7da268b6d1f1E\\\"(ptr sret(%\\\"core::result::Result<std::sync::mutex::MutexGuard<\\'_, i32>, std::sync::poison::PoisonError<std::sync::mutex::MutexGuard<\\'_, i32>>>\\\") %_3, ptr align 4 %safe_x)\", llvm_type: \"void\" } }")

	RESOURCE: Symbol("%safe_x 0xaaaacc70a3b0")

	SECOND_LOCK: Symbol("InstructionValue { instruction_value: Value { name: \"\", address: 0xaaaacc70cf80, is_const: false, is_null: false, is_undef: false, llvm_value: \"  invoke void @\\\"_ZN3std4sync5mutex14Mutex$LT$T$GT$4lock17h8dad7da268b6d1f1E\\\"(ptr sret(%\\\"core::result::Result<std::sync::mutex::MutexGuard<\\'_, i32>, std::sync::poison::PoisonError<std::sync::mutex::MutexGuard<\\'_, i32>>>\\\") %_19, ptr align 4 %safe_x)\\n          to label %bb10 unwind label %cleanup\", llvm_type: \"void\" } }")
__________
```

### Challenges
This turned out to be quite a challenging project owing primarily to the following reasons:
- Rust function mangling and pointer aliasing is quite difficutly to parse.
- `inkwell` is primarily intended for building LLVM IR and not for analyzing it. Notable shortcomings are...
	- No ability to get `CallSites`.
	- No ability to get predecessors.
	- Limited ability to downcast.
- As a result of the above, a large portion LLVM logic needed to be written as string parsing operations.

### Future Work
- We would like to extend the analysis to detect deadlocks in more complex programs. Testing here has not been extensive.
- We would like to implement the analysis for other synchronization primitives such as `std::sync::RwLock`.
- We would like to implement a rust toolchain for the analysis.