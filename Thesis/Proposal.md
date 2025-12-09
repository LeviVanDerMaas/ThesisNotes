
# Proposal
## Title
Integrating a Dynamic Runtime SIMD Dispatch Mechanism into the FINUFFT Library
*More fancy and not direct FINUFFT mention maybe:* 
Accelerating Non-Uniform Fast Fourier Transforms With a Runtime SIMD Dispatch Mechanism*

# Summary Project Proposal and Problem Definition
The performance of non-uniform fast Fourier transform libraries can be greatly enhanced by leveraging SIMD extensions on CPU's, but CPUs differ widely in the instruction set extensions they support. For example, a binary compiled with AVX2 instructions cannot run on CPUs that do not support at least that extension. Meanwhile a binary compiled with SSE instructions, a predecessor to AVX2, can be executed on CPUs that support AVX2,  but such a binary would not utilize the full potential offered by such CPUs.

To resolve this portability-performance trade-off in the Flatiron Institute Non-Uniform Fast Fourier Transormation Library (FINUFFT), this project aims to integrate a runtime SIMD dispatch mechanism into FINUFFT using the xsimd library. Multiple SIMD variants of certain functionality will be compiled into the binary, and the library will automatically select the fastest supported variant at runtime. This approach preserves portability while ensuring the library automatically exploits the hardware’s full capabilities.

# Intended Outcomes
Integration of a fully functional, performant, and automatic runtime SIMD dispatch mechanism into FINUFFT that does not sacrifice portability, and an evaluation demonstrating the improved performance that this leads to compared to the current baseline.

# Intended Milestones
- Initial implementation of a xsimd-based runtime SIMD dispatch mechanism in a toy project, to get familiar.
- At least a fully functional prototype implementation of the proposed mechanism into the FINUFFT library
- After that, preferably a fully polished implementation that can be merged into FINUFFT (mostly) directly.