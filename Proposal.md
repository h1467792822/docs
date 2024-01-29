# Proposal

Added an optimization option that allows users to replace full symbol mangling names based on hash digests, greatly reducing the length of symbol names in dylib. At the expense of commissioning capabilities such as readability of symbol names, this option eliminates the space bottleneck encountered by using Rust to replace existing C/C++ functional modules in resource-constrained scenarios.

# Motivation

The average length of symbol names in the rust standard library is about 100 bytes, while the average length of symbol names in the C++ standard library is about 50 bytes. In some embedded environments where dynamic library are widely used, rust dynamic library symbol name space has become one of the key bottlenecks of application, Especially when the existing C/C++ module is reconstructed into the rust module.

The standard library is a typical example. The proportion of the `.dynstr` segment in the entire elf file in the standard library of rust and that in the standard library of c++ is compared. Compare the data of specific symbols in `.dynstr`. The comparison data is as follows:

The proportion of `.dynstr` in the rust standard library is about twice that in C++:

||`.dynstr`|total||
|---|---|---|---|
|rust libstd.so|265559|804896|0.33|
|rust libstd.so(`symbol_mangling_version=v0`)|318710|858144|0.37|
|c++ libstdc++.so|267295|1594864|0.17|

**Remarks**:

1. The build environment is ubuntu 18.04 LTS.
1. The rust standard library build options include: `panc="abort", opt-leve="z", codegen-units=1,strip=true, debug=true`. and the `.rustc` section is removed.

In C++, the average length of symbol names after mangling is about 50, while in rust, the length of symbol names after mangling is about 100.

|||size|count|average|
|---|---|---|---|---|
|rust libstd.so|start with `_ZN`|263106|2722|96|
||not start with `_ZN`|2314|184|12|
|rust libstd.so(`symbol_mangling_version=v0`)|start with `_R`|316371|2722|116|
||not start with `_R`|2314|184|12|
|c++ libstdc++.so|start with `_ZN`|218957|4187|52.3|
||not start with `_ZN`|49331|1362|36.2|

Finding a way to shorten the symbolic names of rust dynamic libraries is of great value.

# Design

## Shorter symbolic names based on digests

**The solution is to replace its full mangling name with a digest**, select a specific hash algorithm to generate a digest from the full symbolic mangling name. and the space of the `.dynstr` section can be greatly reduced, even better than that of C++.

We can use post-processing tools to do this, right? For example, `objcopy`. Unfortunately, `objcopy --redefine-syms` cannot modify or shorten the symbol name of `.dynsym`. Using post-processing tools to reduce dynstr segment space is much more difficult than expected. **If rustc itself can solve the problem of using rust language in specific scenarios**, it will be the simplest and most convenient solution for users and will greatly promote the application of rust language in a wider range of scenarios.

## Usage Constraints

For debugging, If you replace the full symbol name with the digest, it is difficult to find the corresponding code based on the symbol name of the dynamic library. Therefore, the debugging information backed up by the user and the full code are required. Considering that crate is widely used in rust, **the final symbolic name consists of crate and a digest  is a reasonable scheme**.

What can I do if a symbol name conflict occurs due to a hash conflict? After all, hash conflicts are theoretically unavoidable. There are two scenarios for this conflict, one is inside the dylib and the other is between multiple dylibs.
1. If the scene is inside dylib, rustc will find and report this problem. The user can choose not to optimize the symbol name length, or the **internal hash algorithm allows the user to provide a new salt value for recalculation**, which allows the user to try to eliminate such a small probability of collision events. 
> - If an rlib is dependent on a dylib, all symbols of public APIs in the rlib will be included in the dylib. If there is a hash conflict between these symbols, they will be detected immediately. 
> 
> - The case for generic functions is different. The symbol of a generic function may be scattered in multiple downstream dylibs. If the symbol of a generic function still contains `crate name`, hash conflicts between the generic function and other symbols of the same `crate` cannot be detected in time during construction. This symbol conflict is left over until it occurs during run time. In this case, **`instantiating-crate name` is used to replace `crate name`** can completely eliminate the risk of the preceding potential hash conflict.
2. In a scenario between dylibs, because the final symbol name already includes the name of crate, Symbol conflicts may occur only when different versions of the same rlib are depended on by different dylibs. The same symbol name corresponds to implementations of different versions. Generally, if an rlib is used by multiple dylibs, **the rlib should be replaced by its dylib**. If multiple dylibs depend on the same rlib for other reasons, the symbols in the upstream rlib are not expected to be exported. This is similar to the `-Wl, --exclude-libs` function of gcc, In rust, `-C link-arg=-Wl,--exclude-libs=libfoo.rlib` can be used to avoid exporting symbols in the upstream rlib.

> By comparing the exported symbols of each dynamic library, users are able to detect symbol conflicts between dylibs in advance. Although symbol name conflicts between dynamic libraries may lead to undefined behavior during running, the risk is controllable in actual application after users know the constraint.

In addition,**it is not compatible with existing options: `-C instrument-coverage`**.

## Final Design Scheme

The value `hashed` of `symbol-mangling-version` is added to support shortening symbol names.

1. Currently, only the unstable option: `-C symbol-mangling-version=hashed -Z unstable-options` can be used.
1. For non-generic functions，the format of the final symbol name is `_RNxC{length}{crate name}{length}H{64-bits hash}`. For generic functions，the format of the final symbol name is `_RNxC{length}{instantating-crate name}{length}H{64-bits hash}`.  complies with the existing specification (https://rust-lang.github.io/rfcs/2603-rust-symbol-name-mangling-v0.html#syntax-of-mangled-names).
1. The 64-bit hash is encoded based on `base-62` and the final terminator `_` is removed because it does not help prevent hash collisions.
1. The salt value can be transferred using `-C metadata=<salt>`  to eliminate rare hash conflicts.

# Test Data

According to the test data, the total space of the entire dylib is saved by about 20% when this option is used. For details, see the PR: https://github.com/rust-lang/rust/pull/118636
