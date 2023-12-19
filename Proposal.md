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

Can a new mangling rule solve this problem? Almost impossible. Because rust's package management mechanism, modular code organization form, and flexible trait mechanism are technical advantages over C/C++, a full mangling name is longer than C/C++ is an inevitable product of these technical advantages. 

**The solution is to replace its full mangling name with a digest**, select a specific hash algorithm to generate a digest from the full symbolic mangling name. and the space of the `.dynstr` section can be greatly reduced, even better than that of C++.

We can use post-processing tools to do this, right? For example, `objcopy`. Unfortunately, `objcopy --redefine-syms` cannot modify or shorten the symbol name of `.dynsym`. Using post-processing tools to reduce dynstr segment space is much more difficult than expected. **If rustc itself can solve the problem of using rust language in specific scenarios**, it will be the simplest and most convenient solution for users and will greatly promote the application of rust language in a wider range of scenarios.

## Usage Constraints

For debugging, If you replace the full symbol name with the digest, it is difficult to find the corresponding code based on the symbol name of the dynamic library. Therefore, the debugging information backed up by the user and the full code are required. Considering that crate is widely used in rust, **the final symbolic name consists of crate and a digest  is a reasonable scheme**.

What can I do if a symbol name conflict occurs due to a hash conflict? After all, hash conflicts are theoretically unavoidable. There are two scenarios for this conflict, one is inside the dylib and the other is between multiple dylibs.

1. If the scene is inside dylib, rustc will find and report this problem. The user can choose not to optimize the symbol name length, or the **internal hash algorithm allows the user to provide a new salt value for recalculation**, which allows the user to try to eliminate such a small probability of collision events.
1. In a scenario between dylibs, because the final symbol name already includes the name of crate, the conflict occurs only when the same rlib is depended on by multiple dylibs. Generally, if an rlib is used by multiple dylibs, the rlib should be replaced by its dylib. If multiple dylibs depend on the same rlib for other reasons, the symbols in the upstream rlib are not expected to be exported. This is similar to the `-Wl, --exclude-libs` function of gcc, this scenario is not discussed here. **You can specify the Crate that can optimize the symbol name**. In this way, if the symbol conflict cannot be eliminated by adjusting the salt value, you can disable the function of optimizing the symbol name length for a specific Crate.

By comparing the exported symbols of each dynamic library, users are able to detect symbol conflicts between dylibs in advance. Although symbol name conflicts between dynamic libraries may lead to undefined behavior during running, the risk is controllable in actual application after users know the constraint.

In addition,it is not compatible with existing options: `-C instrument-coverage`.

Finally, **the new option and its parameter values are not written into the binary file**. You must ensure that the configuration of the option in the dylib build project is consistent with that in the build project that depends on the dylib.

## Final Design Scheme

Added an optimization option, `symbol_mangling_digest`, to post-process the symbol  maingling name in rustc. The main functions include: 
1. Allows users to specify the crate name or prefix to determine the optimization scope.
1. Allows users to specify salt values that can be used for hash calculation. 
1. providing different policies to generate new symbol name, including crate name and a hash-calculated digest; 

The new option are defined as follows: `-Z symbol_mangling_digest=<crate name>[*],...[,excluded=<true|false>][,salt=<value>][,level=<1|2>]`

- `crate_name[*],...`: Name of a crate. Multiple crate names are allowed. If the suffix `*` is carried, it is the prefix of the crate name. It and `excluded` together determine the range of symbols to be optimized. Users must be very clear about the optimization range. If the crate supports regular expression matching, the optimization range is difficult to determine. May cause confusion. Defaults to null.
- `excluded=<true|false>`: If the value is `false`, only the names of symbols whose crate names are successfully matched are optimized. If the value is `true`, it indicates that the name of the symbol that fails to be matched is optimized. The default value is `false`.
- `salt=<value>`: User-specified salt used in hash calculation. The default value is null.
- `level=<1|2>`: Specifies the combination policy of the final symbol name. If the value is `1`, the final combination format is `{crate}.{item}.{hash32}` . If the value is `2`, the final combination format is `{crate}.{hash64}`. The default value is `2`. 
>
>In the libstd.so, the number of symbols included in different crates is as follows:
>
>|crate|count|
>|---|---|
>|std|1121|
>|core|1301|
>|alloc|711|
>
>The default `level=2` using `hash64` is an option that takes into account the final symbol length and hash conflict.
>
>`level=1` is used to further reduce the possible conflict scope by using `item name`, and the readability of dynamic library symbol names is better. Data based on libstd.so shows that its final symbol name length is generally longer than `level=2`.

# Test Data

According to the test data, the total space of the entire dylib is saved by about 20% when this option is used. For details, see the PR: https://github.com/rust-lang/rust/pull/118636
