Where we want to implement dynamic SIMD dispatch is here:
https://github.com/flatironinstitute/finufft/blob/193d4f7c2549eab754666c59c419253085beb6e0/src/c_interface.cpp#L127.

The main workflow that this file cocerns is generally:
- Create a plan (finufft_makeplan I think)
- Pass to finufft_setpts
- Pass to one finufft_execute

Advice would be to expand the *FINUFFT_PLAN_T* template to also be a template on the architecture type besides just the data type.
**NOTE** Expected difficulties in `finufft/include/finufft/simd.hpp` will be at line 61, in that you must also account for the architecture being passed because it will otherwise be overriden!

You should look at the makefile and `src/Cmakelist.txt`

# Running tests
It seems you need to run the test/.check_finufft.sh script inside of build/dev/test for it to actually work properly (and you'll need to mkdir a results dir for it to work correctly).


# Interfaces
It seems that, besides various language and cuda bindings, there are two ["primary" interfaces](https://finufft.readthedocs.io/en/latest/cex.html) to FINUFFT.
- The *C interface*, which actually serves as both the C++ interface and C interface, as it is designed to be C++ compatible. It is a "simple interface" that handles most basic uses cases of NUFFT.
- The *Guru interface*, which offers more flexiblity, this concerns things like `finufft_makeplan`, `finufft_setpts`, `finufft_execute`.
Both are located in `src/c_interface.cpp`, with the C interface just invoking the Guru Interface by calling through a helper-layer (located in a local anonymous namespace).