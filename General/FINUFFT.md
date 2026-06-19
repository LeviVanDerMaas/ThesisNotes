Where we want to implement dynamic SIMD dispatch is here:
https://github.com/flatironinstitute/finufft/blob/193d4f7c2549eab754666c59c419253085beb6e0/src/c_interface.cpp#L127.

You should look at the makefile and `src/Cmakelist.txt`

The main workflow that this file cocerns is generally:
- Create a plan (finufft_makeplan I think)
- Pass to finufft_setpts
- Pass to one finufft_execute
- Think you also need to call finufft_delete at some point to free memory

Advice would be to expand the *FINUFFT_PLAN_T* template to also be a template on the architecture type besides just the data type. The header for *FINUFFT_PLAN_T* is in in `include/finufft/plan.hpp`, but its constructror source is in `include/finufft/makeplan.hpp`.
**NOTE** Expected difficulties in `finufft/include/finufft/simd.hpp` will be at line 61, in that you must also account for the architecture being passed because it will otherwise be overriden!

# Build Toolchain
## Building
For *dev* preset, we need **cmake(curses) and ninja.**  For *all* we also need **cudatoolkit**.
The **clang-tools** suite might also be nice for various dev features.
## Running tests
It seems you need to run the test/.check_finufft.sh script inside of build/dev/test for it to actually work properly (and you'll need to mkdir a results dir for it to work correctly).
## Clangd/LSP
Post-process the generated `compile_commands.json` with something like **compdb** (e.g. `compdb -p build/dev list > compile_commands.json`), otherwise it will have several issues. I presume this is because build systems like cmake generally don't output `compile_commands.json` entries for header files, and clangd will then just use heuristics for it; tools like **compdb** add these entries. Also if you still seem to be missing or getting errors inside certain "core-ish" headers (like *omp.h* or *intrinsics defintions*) then that's probably because it found the gcc version of those headers instead of the clang version; on Nix you should be able to resolve that by just including the clang versions of those in a shell.

In a pinch **VSCode** with the *C/C++ extension* should also be able to help you out with figuring things. Do make sure to launch it inside a properly configured *nix-shell* though.



# Codebase
## Interfaces
It seems that, besides various language and cuda bindings, there are two ["primary" interfaces](https://finufft.readthedocs.io/en/latest/cex.html) to FINUFFT. Both can be called from both C++ and C via finufft.h
- The *Guru interface*, which offers most flexiblity, this concerns things like `finufft_makeplan`, `finufft_setpts`, `finufft_execute`. They essentially allow you to precisely construct your own calculation. The actual computational logic appaers to be done via *finufft_execute*, which goes through the plan's own defined `execute` method which defers to `execute_internal` in `include/finufft/execute`.
- The *C interface*, which actually serves as both the C++ interface and C interface, as it is designed to be C compatible. It is a "simple interface" that handles most basic uses cases of NUFFT.  All of the the functions under this category just wrap calls to the Guru Interface: they do this by eventually invoking `safe_finufft_call` from  safe_call.h
- In both of these, each function has both a double and single precision version and these are respectively prefixed with `finufft` and `finufftf`.
- All go through `safe_finufft_call`, which takes a lambda and arguments and perfect forwards those arguments to the lambda: this wrapper ensures the C++ callables are safe to use in C by handling some edge cases and things like exceptions.
Our main interest is thus the Guru Interface since these are the actual entrypoints into finufft.

## XSIMD and SIMD usage in the current codebase
References to xsimd seem to be made in:
- include/finufft/{simd, spread, interp}.hpp: various instructions on batches. X
- src/fft,  and include/finufft/{plan, execute}.hpp: only for the aligned allocator. I think this is where the transform actually happens? with spread combing before and interp after.
- devel: probably not relevant for us
- **ALL of these include simd.hpp in some way**

There is also one instance of an *xsimd::best_arch*, inside find_optimal_simd_widht().

It appears that throughout the code base, currently *xsimd::make_sized_batch* is used in combination with logic to find the largest possible simd size that the host architecture supports, in order to obtain xsimd::batches and other simd info with the appropriate architecture. Notably it appears that *PaddedSIMD*, a type alias set in `simd.hpp` that relies on `make_sized_batch` and some other constexpr function that eventually make use of the aformentioned *xsimd::best_arch* to find the best simd size,  is used at various point in *interp.hpp* and *simd.hpp* to derive the batch type (usually referred to with alias `simd_type`) to be used.  *spread.hpp* also relies on *PaddedSIMD*, as it instead derives `simd_type` from struct `KernelBufferLayout` which is defined in *simd.hpp* again. 

It is also interesting to note that `makeplan.hpp` uses *get_padded_simd_width* to set a variable named `simd_size`, but that variable appears to be unused beyond a debug log message.