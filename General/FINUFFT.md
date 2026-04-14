Where we want to implement dynamic SIMD dispatch is here:
https://github.com/flatironinstitute/finufft/blob/193d4f7c2549eab754666c59c419253085beb6e0/src/c_interface.cpp#L127.

The main workflow that this file cocerns is generally:
- Create a plan (finufft_makeplan I think)
- Pass to finufft_setpts
- Pass to one finufft_execute

Advice would be to expand the *FINUFFT_PLAN_T* template to also be a template on the architecture type besides just the data type.
**NOTE** Expected difficulties in `finufft/include/finufft/simd.hpp` will be at line 61, in that you must also account for the architecture being passed because it will otherwise be overriden!

You should look at the makefile and `src/Cmakelist.txt`