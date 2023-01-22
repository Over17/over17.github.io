---
layout: post
title:  "Arm outline atomics"
date:   2023-01-22 13:36:00 +0100
categories: performance atomics arm
---
# Arm outline atomics

One of the features introduced in the Armv8.1-A architecture version were new atomic instructions, also known as LSE (large system extensions). They include a number of atomic operations done with a single atomic instruction (CAS – compare and swap, arithmetic operations etc.) which result in a significant performance improvement over “normal” implementation, which usually has exclusive load, operation, exclusive store, check if the operation was not interrupted and a branch if it was interrupted (also known as LL/SC). The performance improvement could be estimated as roughly 20%, and could benefit lockless algorithms and containers in Unity, as well as the job system (especially with tiny jobs) and other subsystems.

An obvious question which follows is device support. Currently, when targeting Arm64 in Unity, we default to the vanilla Armv8-A architecture version, which is fully supported by every 64-bit Arm device in the world. Support for Armv8.1-A extensions was mandated in Cortex A75, which is equivalent to Samsung Galaxy S9 (released 2018). This means that a significant share of our end-users are still on a device which doesn’t support LSE. If such a device stumbles upon an LSE instruction, the process will immediately die with an “invalid operation” exception (ILLOPC). So we cannot just go and enable LSE all over the place (which can be done by passing `-march=armv8-a+lse` to the compiler).

A potential solution would be to have both options (LSE and LL/SC) with a runtime dispatch. Luckily, Arm has done a fantastic job which takes care of everything in the compiler itself, and it’s called outline atomics (option present in GCC and clang): [https://reviews.llvm.org/D91157](https://reviews.llvm.org/D91157) and [https://github.com/llvm/llvm-project/blob/main/llvm/docs/Atomics.rst](https://github.com/llvm/llvm-project/blob/main/llvm/docs/Atomics.rst). The idea is as follows:
-	You add `-moutline-atomics` flag to your compiler command line
-	The compiler generates code which does runtime detection of your CPU capabilities (does it support LSE or not?)
-	The compiler generates intrinsic functions (for different kinds of atomic operations) which have two branches: LSE and LL/SC, which one is called depends on the CPU capabilities (see previous item)
-	Every time your code does an atomic operation, the compiler inserts a call to the corresponding intrinsic function. This means that on old devices, the path with LL/SC will be selected, while LSE-capable devices will be running LSE code – all that with zero changes needed to your code, nice and backward compatible! You only have to add the compiler flag.

Magic!

But in the end, is it free performance? For every atomic instruction, you have to pay the cost of calling a function and doing an if on CPU capabilities (likely an easy target to the branch predictor). Compare this to inlined atomic instructions – be it LL/SC or pure LSE (if you’re targeting only modern CPUs). Our friends at Arm (including the developer of the outline atomics feature in LLVM) say that the benefit of LSE is much higher than the cost associated with the function call, so it’s a net plus. If so, it sounds like we could provide some performance improvement for our users at a very low cost! Let’s give it a try.

Before we start, I must share [a well-known study of LSE in MySQL](https://mysqlonarm.github.io/ARM-LSE-and-MySQL/) which TL;DR resulted in “no benefit”. Interesting, and let’s keep it in mind.

## Implementation

It’s a one-liner, a usual case for complicated but well-designed features: just add `-moutline-atomics` to the clang flags.

## Performance assessment strategy

This is the most important thing. Here’s the plan:

-	A non-LSE-capable device (an older device)
	-	Baseline: no outline atomics, basically inlined LL/SC code, as it exists currently in our codebase.
	-	Outline atomics: enable outline atomics. In this case, no improvement is expected, just the overhead of having to call the outlined functions. A perfect result would be no regression or a regression around measurement error (up to 5%). If the regression is significant (20+%), then it’s a red flag.
-	An LSE-capable-device (a newer device)
	-	Baseline: no outline atomics, basically inlined LL/SC code, as it exists currently in our codebase.
	-	Outline atomics: enable outline atomics. In this case, an improvement is expected because of the new LSE instructions, minus the function call overhead. A perfect result would be an improvement of 10+%. No improvement is a red flag.
	-	Pure LSE: enable LSE as the target in the compiler command line. This option is there to compare with outline atomics and try to assess the function call and CPU capabilities branch overhead. Not shippable because not compatible with older devices, but helps understand the nature of improvement/regression with outline atomics. Must perform better than previous option.

The following devices were used:
-	non-LSE: nVidia Shields in our CI buildfarm, Cortex A57
-	non-LSE: Huawei Honor 9, Cortex A53 (tested locally)
-	LSE: Samsung Galaxy S22, Cortex X2+A710+A510, latest Armv9 device on the market at the time of writing (tested locally)

## Round 1. Main Unity and CI

We have remarkable Native Performance tests which run on CI and report to a huge database. Let’s run the whole suite with outline atomics and without, and compare the results.

On a Shield, there’s a single change that’s worth mentioning: a slight regression in global_no_contention_Atomic_Add.

![unity_perftest_shield_global_no_contention_Atomic_Add](/assets/images/2023-01-22-arm-outline-atomics-unity_perftest_shield_global_no_contention_Atomic_Add.png) 

Few other regressions are most likely unrelated.

Let’s try on an LSE-capable device then. I’m running the tests locally and report data to the database.

The results are unfortunately quite unstable - and it partly relies on the fact that it’s running on a consumer device lying on my table. However, few results are curious.

![unity_perftest_s22_1](/assets/images/2023-01-22-arm-outline-atomics-unity_perftest_s22_1.png) 
![unity_perftest_s22_2](/assets/images/2023-01-22-arm-outline-atomics-unity_perftest_s22_2.png) 
![unity_perftest_s22_3](/assets/images/2023-01-22-arm-outline-atomics-unity_perftest_s22_3.png)
![unity_perftest_s22_4](/assets/images/2023-01-22-arm-outline-atomics-unity_perftest_s22_4.png)
 

The global_no_contention_Atomic_Add which regressed on a Shield, shows an improvement on S22 (0.12 => 0.10ms). Some of the MemoryManagerPerformance tests improved significantly. Looks good so far!

Okay let’s now go to the code and try to understand what’s going on.

## The Code

Of course, we have more than one atomics implementation in the Unity codebase. :(

-	We have ExtendedAtomicOps-arm64.h. It uses explicit inline assembly which always generates oldschool LDXR/STXR instructions. Outline atomics have no effect on these, and they are used in quite a few places, which explains why there’s less effect to be measured all over the board.
-	We have baselib’s atomics, which are a much more recent, platform-agnostic implementation using compiler instrinsics (GCC builtins, also implemented in LLVM). It is used in MemoryManager and other places, but currently has smaller adoption than the old ExtendedAtomicOps. This implementation benefits from LSE and outline atomics, which explains why there are gains in MemoryManager performance tests.
-	Some places are still using std::atomic, which benefits from LSE/outline atomics too.

The overall strategy is to move as many consumers as possible to baselib, or to rewrite ExtendedAtomicOps to be a shim which redirects to baselib. LSE/outline atomics would benefit that.

However, if no regressions are found, it may be a good idea to adopt outline atomics now – and patiently wait for the future to come. It also means more motivation to move to baselib’s atomics all over the codebase.

Luckily, baselib has a set of benchmarks! Let’s run them and look at the results.

## Round 2. Baselib Benchmarks

Baselib possesses a great collection of benchmarks, which cover all atomics use cases, plus collects the same results for std::atomics to use as a baseline.

The results of running the benchmarks on Galaxy S22 (LSE-capable) are available in the table below.

| Testcase | Benchmark | default, ns | outline, ns | Change |
|---|---|---|---|---|
|Atomic - Single Threaded (int32_t) | load(relaxed)| 0,0003925| 0,000382857| -2,46%|
|Atomic - Single Threaded (int32_t) | load(acquire)| 0,00039125| 0,00039125| 0,00%|
|Atomic - Single Threaded (int32_t) | load(seq_cst)| 0,00039| 0,000392045| 0,52%|
|Atomic - Single Threaded (int32_t) | store(relaxed)| 0,0003925| 0,00039| -0,64%|
|Atomic - Single Threaded (int32_t) | store(release)| 0,00039| 0,00039| 0,00%|
|Atomic - Single Threaded (int32_t) | store(seq_cst)| 0,00039065| 0,00039| -0,17%|
|Atomic - Single Threaded (int32_t) | fetch_add(relaxed)| 5,67388| 4,99653| -11,94%|
|Atomic - Single Threaded (int32_t) | fetch_add(acquire)| 5,6747| 5,04597| -11,08%|
|Atomic - Single Threaded (int32_t) | fetch_add(release)| 5,67251| 4,99649| -11,92%|
|Atomic - Single Threaded (int32_t) | fetch_add(acq_rel)| 5,69118| 4,97058| -12,66%|
|Atomic - Single Threaded (int32_t) | fetch_add(seq_cst)| 7,31765| 7,44496| 1,74%|
|Atomic - Single Threaded (int32_t) | fetch_and(relaxed)| 5,7294| 4,961| -13,41%|
|Atomic - Single Threaded (int32_t) | fetch_and(acquire)| 5,67328| 4,95233| -12,71%|
|Atomic - Single Threaded (int32_t) | fetch_and(release)| 5,72666| 4,99676| -12,75%|
|Atomic - Single Threaded (int32_t) | fetch_and(acq_rel)| 5,67334| 4,99625| -11,93%|
|Atomic - Single Threaded (int32_t) | fetch_and(seq_cst)| 7,34077| 7,48325| 1,94%|
|Atomic - Single Threaded (int32_t) | fetch_or(relaxed)| 5,67699| 4,96993| -12,45%|
|Atomic - Single Threaded (int32_t) | fetch_or(acquire)| 5,67184| 4,99722| -11,89%|
|Atomic - Single Threaded (int32_t) | fetch_or(release)| 5,73154| 4,95138| -13,61%|
|Atomic - Single Threaded (int32_t) | fetch_or(acq_rel)| 5,67594| 4,99676| -11,97%|
|Atomic - Single Threaded (int32_t) | fetch_or(seq_cst)| 7,31725| 7,46938| 2,08%|
|Atomic - Single Threaded (int32_t) | fetch_xor(relaxed)| 5,67038| 5,0441| -11,04%|
|Atomic - Single Threaded (int32_t) | fetch_xor(acquire)| 5,67334| 4,99698| -11,92%|
|Atomic - Single Threaded (int32_t) | fetch_xor(release)| 5,67444| 4,95283| -12,72%|
|Atomic - Single Threaded (int32_t) | fetch_xor(acq_rel)| 5,67227| 4,92638| -13,15%|
|Atomic - Single Threaded (int32_t) | fetch_xor(seq_cst)| 7,2775| 7,45118| 2,39%|
|Atomic - Single Threaded (int32_t) | exchange(relaxed)| 0,00038759| 0,000394176| 1,70%|
|Atomic - Single Threaded (int32_t) | exchange(acquire)| 5,48296| 5,04679| -7,96%|
|Atomic - Single Threaded (int32_t) | exchange(release)| 0,00039125| 0,00039| -0,32%|
|Atomic - Single Threaded (int32_t) | exchange(acq_rel)| 5,43788| 5,0526| -7,09%|
|Atomic - Single Threaded (int32_t) | exchange(seq_cst)| 7,25044| 7,49777| 3,41%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(relaxed. relaxed)| 5,25713| 5,3991| 2,70%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(relaxed. relaxed)| 5,68955| 5,42444| -4,66%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(acquire. relaxed)| 5,24012| 5,39963| 3,04%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(acquire. relaxed)| 5,73798| 5,40916| -5,73%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(acquire. acquire)| 5,23611| 5,39846| 3,10%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(acquire. acquire)| 5,73893| 5,47375| -4,62%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(release. relaxed)| 5,25698| 5,39858| 2,69%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(release. relaxed)| 5,6121| 5,39765| -3,82%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(acq_rel. relaxed)| 5,27817| 5,39882| 2,29%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(acq_rel. relaxed)| 5,7372| 5,45335| -4,95%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(acq_rel. acquire)| 5,23561| 5,39926| 3,13%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(acq_rel. acquire)| 5,66822| 5,40713| -4,61%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(seq_cst. relaxed)| 5,26859| 6,82747| 29,59%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(seq_cst. relaxed)| 7,50258| 8,26107| 10,11%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(seq_cst. acquire)| 5,28095| 6,78222| 28,43%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(seq_cst. acquire)| 7,48468| 8,25413| 10,28%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak fail(seq_cst. seq_cst)| 5,2449| 7,58497| 44,62%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_weak success(seq_cst. seq_cst)| 5,6967| 7,65822| 34,43%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(relaxed. relaxed)| 5,23057| 5,39805| 3,20%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(relaxed. relaxed)| 5,80849| 5,46938| -5,84%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(acquire. relaxed)| 5,21846| 5,39832| 3,45%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(acquire. relaxed)| 5,78097| 5,42467| -6,16%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(acquire. acquire)| 5,25948| 5,39944| 2,66%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(acquire. acquire)| 5,76221| 5,39818| -6,32%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(release. relaxed)| 5,20002| 5,39859| 3,82%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(release. relaxed)| 5,79507| 5,36288| -7,46%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(acq_rel. relaxed)| 5,2727| 5,3982| 2,38%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(acq_rel. relaxed)| 5,7404| 5,40017| -5,93%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(acq_rel. acquire)| 5,17358| 5,39925| 4,36%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(acq_rel. acquire)| 5,79372| 5,35798| -7,52%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(seq_cst. relaxed)| 5,22951| 6,78288| 29,70%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(seq_cst. relaxed)| 7,49661| 8,26927| 10,31%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(seq_cst. acquire)| 5,28694| 6,82693| 29,13%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(seq_cst. acquire)| 7,53192| 8,25459| 9,59%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong fail(seq_cst. seq_cst)| 5,1914| 7,51776| 44,81%|
|Atomic - Single Threaded (int32_t) | cmp_xchg_strong success(seq_cst. seq_cst)| 7,50535| 7,64803| 1,90%|
|std::atomic - Single Threaded (int32_t) | load(relaxed)| 0,00039125| 0,00039125| 0,00%|
|std::atomic - Single Threaded (int32_t) | load(acquire)| 0,00039| 0,00039| 0,00%|
|std::atomic - Single Threaded (int32_t) | load(seq_cst)| 0,000396331| 0,00039| -1,60%|
|std::atomic - Single Threaded (int32_t) | store(relaxed)| 0,0003925| 0,0003925| 0,00%|
|std::atomic - Single Threaded (int32_t) | store(release)| 0,000393404| 0,00039| -0,87%|
|std::atomic - Single Threaded (int32_t) | store(seq_cst)| 0,00039| 0,000389173| -0,21%|
|std::atomic - Single Threaded (int32_t) | fetch_add(relaxed)| 5,7106| 5,15507| -9,73%|
|std::atomic - Single Threaded (int32_t) | fetch_add(acquire)| 5,72031| 5,26461| -7,97%|
|std::atomic - Single Threaded (int32_t) | fetch_add(release)| 5,66582| 5,21962| -7,88%|
|std::atomic - Single Threaded (int32_t) | fetch_add(acq_rel)| 5,6786| 5,22675| -7,96%|
|std::atomic - Single Threaded (int32_t) | fetch_add(seq_cst)| 5,71181| 5,17488| -9,40%|
|std::atomic - Single Threaded (int32_t) | fetch_and(relaxed)| 5,76524| 5,10852| -11,39%|
|std::atomic - Single Threaded (int32_t) | fetch_and(acquire)| 5,66264| 5,2652| -7,02%|
|std::atomic - Single Threaded (int32_t) | fetch_and(release)| 5,6594| 5,21986| -7,77%|
|std::atomic - Single Threaded (int32_t) | fetch_and(acq_rel)| 5,71097| 5,17591| -9,37%|
|std::atomic - Single Threaded (int32_t) | fetch_and(seq_cst)| 5,7106| 5,22749| -8,46%|
|std::atomic - Single Threaded (int32_t) | fetch_or(relaxed)| 5,71032| 5,15279| -9,76%|
|std::atomic - Single Threaded (int32_t) | fetch_or(acquire)| 5,71144| 5,31691| -6,91%|
|std::atomic - Single Threaded (int32_t) | fetch_or(release)| 5,67889| 5,25348| -7,49%|
|std::atomic - Single Threaded (int32_t) | fetch_or(acq_rel)| 5,76721| 5,17559| -10,26%|
|std::atomic - Single Threaded (int32_t) | fetch_or(seq_cst)| 5,71097| 5,17544| -9,38%|
|std::atomic - Single Threaded (int32_t) | fetch_xor(relaxed)| 5,76692| 5,17489| -10,27%|
|std::atomic - Single Threaded (int32_t) | fetch_xor(acquire)| 5,71116| 5,26452| -7,82%|
|std::atomic - Single Threaded (int32_t) | fetch_xor(release)| 5,65957| 5,27959| -6,71%|
|std::atomic - Single Threaded (int32_t) | fetch_xor(acq_rel)| 5,76536| 5,17506| -10,24%|
|std::atomic - Single Threaded (int32_t) | fetch_xor(seq_cst)| 5,65957| 5,17515| -8,56%|
|std::atomic - Single Threaded (int32_t) | exchange(relaxed)| 0,550552| 0,720635| 30,89%|
|std::atomic - Single Threaded (int32_t) | exchange(acquire)| 5,84268| 5,26468| -9,89%|
|std::atomic - Single Threaded (int32_t) | exchange(release)| 0,55052| 1,78422| 224,10%|
|std::atomic - Single Threaded (int32_t) | exchange(acq_rel)| 5,83989| 5,26444| -9,85%|
|std::atomic - Single Threaded (int32_t) | exchange(seq_cst)| 5,9977| 5,22013| -12,96%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(relaxed. relaxed) fail| 5,46889| 9,06492| 65,75%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(relaxed. relaxed) success| 5,58666| 6,69744| 19,88%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acquire. relaxed) fail| 4,9988| 9,16198| 83,28%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acquire. relaxed) success| 5,6649| 7,05545| 24,55%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acquire. acquire) fail| 5,20166| 9,24713| 77,77%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acquire. acquire) success| 5,68584| 7,02827| 23,61%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(release. relaxed) fail| 5,10529| 9,02168| 76,71%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(release. relaxed) success| 5,78219| 8,04999| 39,22%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acq_rel. relaxed) fail| 4,93453| 9,24931| 87,44%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acq_rel. relaxed) success| 5,67454| 8,01747| 41,29%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acq_rel. acquire) fail| 5,26328| 9,16154| 74,07%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(acq_rel. acquire) success| 5,64472| 8,09841| 43,47%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. relaxed) fail| 5,35597| 9,17655| 71,33%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. relaxed) success| 5,77502| 8,03009| 39,05%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. acquire) fail| 5,24321| 9,2467| 76,36%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. acquire) success| 5,7113| 8,04795| 40,91%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. seq_cst) fail| 5,12202| 9,17788| 79,18%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_weak(seq_cst. seq_cst) success| 5,62336| 8,02618| 42,73%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(relaxed. relaxed) fail| 5,31224| 9,06184| 70,58%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(relaxed. relaxed) success| 5,55487| 6,63493| 19,44%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acquire. relaxed) fail| 5,07827| 9,30801| 83,29%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acquire. relaxed) success| 5,56841| 7,07708| 27,09%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acquire. acquire) fail| 5,43704| 9,27766| 70,64%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acquire. acquire) success| 5,70926| 6,96854| 22,06%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(release. relaxed) fail| 5,14328| 9,10133| 76,96%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(release. relaxed) success| 5,75052| 8,08084| 40,52%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acq_rel. relaxed) fail| 5,05983| 9,29759| 83,75%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acq_rel. relaxed) success| 5,6966| 8,09653| 42,13%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acq_rel. acquire) fail| 5,43815| 9,31471| 71,28%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(acq_rel. acquire) success| 5,79364| 8,02073| 38,44%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. relaxed) fail| 5,42037| 9,30762| 71,72%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. relaxed) success| 5,78459| 8,03016| 38,82%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. acquire) fail| 5,1899| 9,25522| 78,33%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. acquire) success| 5,70771| 8,05765| 41,17%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. seq_cst) fail| 5,11089| 9,39751| 83,87%|
|std::atomic - Single Threaded (int32_t) | cmp_xchg_strong(seq_cst. seq_cst) success| 5,6496| 8,02999| 42,13%|

Most important observations are:
-	baselib’s fetch_[op], exchange, compare-and-exchange show a clear improvement of some 10-20% - which is expected – and is a great result!
-	Operations with SEQ_CST memory model show a much worse improvement and even regressions in some cases! Needs investigation.
-	std::atomic has significantly regressed in compare_exchange_weak and compare_exchange_strong operations.

## Round 3. Mandatory assembly check!

Okay, now let’s try to find out what’s happening behind the scenes and try to explain the performance results on SEQ_CST operations.

Here’s an example outline-atomics function:

```
00000000007a3d60 <__aarch64_cas4_acq_rel>:
  7a3d60: 5f 24 03 d5  	hint	#34
  7a3d64: 10 01 00 90  	adrp	x16, 0x7c3000 <__aarch64_cas8_acq+0x4>
  7a3d68: 10 42 43 39  	ldrb	w16, [x16, #208]
  7a3d6c: 70 00 00 34  	cbz	w16, 0x7a3d78 <__aarch64_cas4_acq_rel+0x18>
  7a3d70: 41 fc e0 88  	casal	w0, w1, [x2]
  7a3d74: c0 03 5f d6  	ret
  7a3d78: f0 03 00 2a  	mov	w16, w0
  7a3d7c: 40 fc 5f 88  	ldaxr	w0, [x2]
  7a3d80: 1f 00 10 6b  	cmp	w0, w16
  7a3d84: 61 00 00 54  	b.ne	0x7a3d90 <__aarch64_cas4_acq_rel+0x30>
  7a3d88: 41 fc 11 88  	stlxr	w17, w1, [x2]
  7a3d8c: 91 ff ff 35  	cbnz	w17, 0x7a3d7c <__aarch64_cas4_acq_rel+0x1c>
  7a3d90: c0 03 5f d6  	ret 
```

It has the branch on CPU capabilities (cbz), LSE implementation (casal) and LL/SC (ldaxr/stlxr).

The call site looks something like this:

```
  4f4170: c0 fd 9f 52  	mov	w0, #65518
  4f4174: c1 fd 9f 52  	mov	w1, #65518
  4f4178: e2 b3 04 91  	add	x2, sp, #300
  4f417c: e0 ff af 72  	movk	w0, #32767, lsl #16
  4f4180: e1 ff af 72  	movk	w1, #32767, lsl #16
  4f4184: f7 be 0a 94  	bl	0x7a3d60 <__aarch64_cas4_acq_rel>
  4f4188: bf 3b 03 d5  	dmb	ish 
```

Highlighted is the full memory barrier generated by baselib when SEQ_CST is used. The corresponding code in baselib has this curious comment:

```
// Patch gcc and clang intrinsics to achieve a sequentially consistent barrier.
// As of writing Clang 9, GCC 9 none of them produce a seq cst barrier for load-store operations.
// To fix this we switch load store to be acquire release with a full final barrier.
```

However, after a lengthy discussion afterwards with folks who wrote the code and our friends at Arm, as well as reading up on the research papers in the internets [https://plv.mpi-sws.org/scfix/paper.pdf](https://plv.mpi-sws.org/scfix/paper.pdf) and [https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0668r5.html](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0668r5.html), [https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/armv8-sequential-consistency](https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/armv8-sequential-consistency), we came to a conclusion that the full memory barrier is too conservative for this case. The full memory barrier is needed for legacy `__sync` GCC builtins though. We’ll be fixing the SEQ_CST atomics in baselib by removing the barrier and re-measuring performance of outline atomics, if time permits.

Here’s the assembly from LL/SC atomic version to compare:

```
  4fbb28: 09 fd 5f 88  	ldaxr	w9, [x8]
  4fbb2c: 3f 01 19 6b  	cmp	w9, w25
  4fbb30: 61 00 00 54  	b.ne	0x4fbb3c 
  4fbb34: 17 fd 09 88  	stlxr	w9, w23, [x8]
  4fbb38: 02 00 00 14  	b	0x4fbb40
  4fbb3c: 5f 3f 03 d5  	clrex
  4fbb40: bf 3b 03 d5  	dmb	ish 
```

Looks pretty logical and neat.

Why is outline-atomics code slower than pure LL/SC, taken that both have the barrier? It couldn’t be just a function call (branch-and-link) overhead, can it? Let’s jump deeper, into a sampling profiler…

## Round 4. Profiling benchmarks

There are two goals to be achieved with a sampling profiler: first, confirm that LSE path is taken; second, try to find out where most time is spent on.

I’m usually sticking to Arm Mobile Studio (Arm Streamline) as a sampling profiler, but since baselib benchmarks are an ELF application and not an APK, I didn’t find a way to profile it out of box in Streamline.

**TODO:**
UPDATE: got a follow-up from Pete Harris @ Arm:
“It works – a little manual setup, but it's not too painful.
-	Push the binary you want to test and gatord to /data/local/tmp
-	From android device shell in /data/local/tmp:
-	Run setprop security.perf_harden 0
-	Run ./gatord --app <app name> <app command line args>
This will pause and wait for the host tool to connect before starting the app
From host tool:
-	In the "Start" tab select "TCP (advanced)"
-	Select the Android device in the device table
-	Configure counters and "Start" when ready
(The Configure application part of this doesn't work for Android)
Once captured, right click on the capture in the "Streamline Data" tab and select "Analyze". Add the ELF images / symbol files you need and reanalyze.”

So it’s doable in Streamline too, but this time I used Google’s simpleperf bundled in Android NDK instead. The “native program” option -np is specially designed for such cases.

Here’s the answer to the first question:

![profile_simpleperf_cas4_acq_rel](/assets/images/2023-01-22-arm-outline-atomics-profile_simpleperf_cas4_acq_rel.png)
 
Yes, runtime detection is working correctly. Yes, LSE instructions are being used (in this particular case, <unknown> is cas\* - compare-and-swap instruction, simpleperf didn’t enable new extensions when dumping disassembly, `hint #34` is a PAC instruction `paciasp`). Yes, they are quite fast – no clear bottleneck, even if the interrupts were not 100% precise because the microarchitecture prefers to not stop at this or that instruction. No issues found here.

Next, let’s check the profile of the call site.
 
![profile_simpleperf_outline_callsite](/assets/images/2023-01-22-arm-outline-atomics-profile_simpleperf_outline_callsite.png)
 
It is unlikely that subs takes that much CPU time. Maybe the CPU interrupts are one instruction off here, but it could also be a side effect of the full memory barrier that the next instruction has to wait longer than expected. Anyway, most of the time is being spent in the memory barrier and around it, which doesn’t really answer the question – why is it performing slower than a non-LSE version? 

Here’s to the assembly of LL/SC version:
 
![profile_simpleperf_ll_sc](/assets/images/2023-01-22-arm-outline-atomics-profile_simpleperf_ll_sc.png) 
 
_(please disregard the yellow markup)_

This one seems to spend most of its time in subs and very little in the memory barrier - which we could once again attribute to microarchitecture preferences, but this is speculative.

Anyway, there is no clear answer as to where the time is spent in the LSE version compared to the LL/SC version when a full memory barrier is in place. I can only state that the outline-atomics version is slower – maybe it’s related to a sequence of function call => memory barrier => arithmetic with flag setting.

Taken that we later decided to remove the full memory barrier from baselib’s SEQ_CST implementations, this regression becomes much less significant.

## Round 5. STL atomics: weak or strong?

STL’s std::atomic performance is used in baselib as a baseline, but it’s still curious to find out why it has become slower with outline-atomics than without them.

When using LL/SC, the benchmark calls into
`__cxx_atomic_compare_exchange_weak<int>(),` 
but when with -moutline-atomics, for some reason it becomes a strong exchange
`__cxx_atomic_compare_exchange_strong<int>()`
which is significantly slower (almost by half).

First I thought there’s a bug in the benchmark, but no, it’s correct. 

When I remove strong compare-and-exchange from the benchmark, the assembly shows calls into the weak version in STL, which is correct! When there are calls to weak and strong versions of STL atomics, then only strong seems to be used. This hints of identical function folding, but why is it happening?

Since our code is using catch benchmarking, I first thought it’s to blame. Looking at the resulting code after all preprocessors run, I see lambdas

```
holder = [&]()
    {
        T prev = value2;
        memory.compare_exchange_strong(prev, value2, order1, order2);
    };
holder = [&]()
    {
        T prev = value2;
        memory.compare_exchange_weak(prev, value2, order1, order2);
    };
```

so I thought maybe it's lambdas which fold.
However, by tuning input and tests here and there, I was able to dump the assembly of the STL's weak and strong versions with outline enabled.

```
bool std::__ndk1::__cxx_atomic_compare_exchange_strong<int>(std::__ndk1::__cxx_atomic_base_impl<int>*, int*, int, std::__ndk1::memory_order, std::__ndk1::memory_order)
bool std::__ndk1::__cxx_atomic_compare_exchange_weak<int>(std::__ndk1::__cxx_atomic_base_impl<int>*, int*, int, std::__ndk1::memory_order, std::__ndk1::memory_order)
```

**They are identical** (apart from branch addresses)! This answers the question why LLVM folded the two functions into one.
I thought okay, maybe it’s an NDK bug – so I tried the latest (at the time) r25beta4. Same result. The code itself is different from r23, but weak and strong versions in r25 are the same.

Trying to isolate it, I created [this repro in godbolt](https://godbolt.org/z/Wqn5TEKaE).
**The assembly with outline intrinsics is identical in strong and weak versions.**

With LL/SC, the assembly does differ, having stxr vs. stlxr instruction.

I raised the question with Arm compiler folks, and we found out that **the pure LSE versions for weak and strong exchange are exactly the same** (confirmed in godbolt). The outline atomics version just builds upon an LSE version, so they are still the same. Doesn’t explain the regression, the LSE instructions must be faster than LL/SC, regardless weak or strong.

The profile confirms that LSE is actually used:
 
![profile_simpleperf_std_atomic_cas4_acq_rel](/assets/images/2023-01-22-arm-outline-atomics-profile_simpleperf_std_atomic_cas4_acq_rel.png)

Overall, some of baselib’s std::atomic benchmarks show performance regressions in outline atomics vs. LL/SC versions on LSE-capable devices. We are currently working with Arm folks on trying to identify the root cause of the STL issues. **To be updated**

## Round 6. Pure LSE

Let’s find out what is the cost related to the function call overhead and CPU capabilities branch introduced by the outline atomics implementation. To remove the function call overhead, we’re building a binary which has atomics implemented in pure inlined LSE instructions.

At first, I tried obtaining some skills in writing inline assembly in C, but after few hours I realized that I can just compile using `-march=armv8-a+lse`, and the compiler will do the job for me automatically! Of course, such a binary can only run on an LSE-capable device, and will generate an illegal instruction exception on an older device (verified locally).

Now, on to performance. Selected benchmark results are available in the table below.

![perf_pure_lse](/assets/images/2023-01-22-arm-outline-atomics-perf_pure_lse.png)

Results:
-	As expected, pure LSE is always faster than outline atomics
-	Absolute overhead of having to call the intrinsic function with outline atomics and check the boolean containing CPU capabilities lies in the range of 0.2-0.3 ns
-	Relative overhead varies from 3 to 7%.

Overall, the result is expected: the overhead is there, but it’s not very high. The performance benefits LSE brings are definitely higher than that.

## Round 7. Benchmarking outline atomics on old, non-LSE devices

The test device for local tests was Huawei Honor 9 (Cortex A53).

**TODO**
The benchmark results in text form are available here: 

The absolute numbers are significantly slower than on the S22 (more than 2x), which is not surprising.

Comparing the outline version to a “normal” one (inline LL/SC atomics), the following conclusions can be made:
-	Outline atomics are always slower than “normal” builds
-	The absolute overhead of outline atomics lies within the range of 5 to 10 ns (!!) which is much higher than on LSE-capable devices. This applies both to STL’s and baselib’s atomics.
-	The relative overhead is quite high in this case, higher than the 20% threshold suggested at the beginning, and in some cases as high as 50%.

Our friends at Arm did their own measurements and confirmed that the overhead of outline atomics on Cortex-A53 is higher than that on LSE-capable devices, likely caused by less efficient branch prediction.
 
## Conclusion

Applying outline atomics is really very very easy – it’s just a compiler flag. The feature implementation is very appealing because of the backward compatibility it provides.

Results on modern, LSE-capable devices are positive, except for some cases with std::atomics which are being investigated further together with Arm folks.

However, the performance results on older devices are quite disappointing. The overhead is significantly higher than the 20% threshold expected at the beginning.

One of Unity’s core values is Users First, and unfortunately, we cannot sacrifice a large portion of our users (Cortex A53 is still very, very popular) for the benefit of another group. We can wait until the share of non-LSE-capable devices is negligible, or try some other tactics (our own dispatch at a higher level – for example, in lockless collections, job system?).

As a result, we are not adopting outline atomics at the moment. This conclusion is to be reassessed in few years.

Your mileage with the feature may vary depending on your target CPUs. If the share of old in-order cores is low for you, then adopting outline atomics may be a valid trade-off.

## Function Multi-Versioning

Since the approach when the compiler automatically generates the necessary runtime dispatch for you is nice and elegant, and it helps optimize your code for latest Arm architectures, Arm started developing a generic version of this feature, called Function Multi-versioning, and it’s being added to Clang and GCC. Here’s the description: [https://github.com/ARM-software/acle/blob/main/main/acle.md#function-multi-versioning](https://github.com/ARM-software/acle/blob/main/main/acle.md#function-multi-versioning) – check it out if you want to target newer Arm architecture extensions while maintaining backward compatibility!
