---
layout: post
title: Pitfalls of TSC usage
categories:
- [English, OS, Hardware]
tags:
- [perf, kernel, linux, virtualization, hardware]
---

##Latency measurement in user space

While user application developers are working on performance sensitive code, one common requirement is do latency/time
measurement in their code. This kind of code could be temporary code for debug, test or profiling purpose, or permanent
code that could provide performance tracing data in software production mode.

Linux kernel provides gettimeofday() and clock_gettime() system calls for user application high resolution time measurement.
The gettimeofday() is us level, and clock_gettime is ns level. However, the major concerns of these system calls usage are
the additional performance cost caused by calling themselves.

In order to minimize the perf cost of gettimeofday() and clock_gettime() system calls, Linux kernel uses the
vsyscalls(virtual system calls) and VDSOs (Virtual Dynamically linked Shared Objects) mechanisms to avoid the cost
of switching from user to kernel. On x86, gettimeofday() and clock_gettime() could get better performance
due to vsyscalls [kernel patch](https://github.com/torvalds/linux/commit/2aae950b21e4bc789d1fc6668faf67e8748300b7),
by avoiding context switch from user to kernel space. But some other arch still need follow the regular
system call code path. This is really hardware dependent optimization.


##Why using TSC?

Although vsyscalls implementation of gettimeofday() and clock_gettime() is faster than regular system calls, the perf cost
of them is still too high to meet the latency measurement requirements for some perf sensitive application.

The TSC (time stamp counter) provided by x86 processors is a high-resolution counter that can be read with a single
instruction (RDTSC). On Linux this instruction could be executed from user space directly, that means user applications could
use one single instruction to get a fine-grained timestamp (nanosecond level) with a much faster way than vsyscalls.

Following code are typical implementation for rdtsc() api in user space application,

	static uint64_t rdtsc(void)
	{
		uint64_t var;
		uint32_t hi, lo;
	
		__asm volatile
		    ("rdtsc" : "=a" (lo), "=d" (hi));
	
		var = ((uint64_t)hi << 32) | lo;
		return (var);
	}

The result of rdtsc is CPU cycle, that could be converted to nanoseconds by a simple calculation.

<pre>ns = CPU cycles * (ns_per_sec / CPU freq)</pre>

In Linux kernel, it uses more complex way to get a better results,

	/*
	 * Accelerators for sched_clock()
	 * convert from cycles(64bits) => nanoseconds (64bits)
	 *  basic equation:
	 *              ns = cycles / (freq / ns_per_sec)
	 *              ns = cycles * (ns_per_sec / freq)
	 *              ns = cycles * (10^9 / (cpu_khz * 10^3))
	 *              ns = cycles * (10^6 / cpu_khz)
	 *
	 *      Then we use scaling math (suggested by george@mvista.com) to get:
	 *              ns = cycles * (10^6 * SC / cpu_khz) / SC
	 *              ns = cycles * cyc2ns_scale / SC
	 *
	 *      And since SC is a constant power of two, we can convert the div
	 *  into a shift.
	 *
	 *  We can use khz divisor instead of mhz to keep a better precision, since
	 *  cyc2ns_scale is limited to 10^6 * 2^10, which fits in 32 bits.
	 *  (mathieu.desnoyers@polymtl.ca)
	 *
	 *                      -johnstul@us.ibm.com "math is hard, lets go shopping!"
	 */


Finally, the code of latency measurement could be,
	
	start = rdtsc();

	/* put code you want to measure here */

	end = rdtsc();

	cycle = end - start;

	latency = cycle_2_ns(cycle)

In fact, above rdtsc implementation are problematic, and not encouraged by Linux kernel.
The major reason is, TSC mechanism is rather unreliable, and even Linux kernel had the hard time to handle it.

That is why Linux kernel does not provide the rdtsc api to user application. However, Linux kernel does not limit the
rdtsc instruction to be executed at privilege level, although x86 support the setup. That means, there is nothing stopping
Linux application read TSC directly by above implementation, but these applications have to prepare to handle some
strange TSC behaviors due to some known pitfalls.
	
##Known TSC pitfalls

###TSC unsafe hardware

1. TSC increments differently on different Intel processors

	Intel CPUs have 3 sort of TSC behaviors,

	* Invariant TSC

	The invariant TSC will run at a constant rate in all ACPI P-, C-, and T-states. This is the architectural behavior
	moving forward. Invariant TSC only appears on Nehalem-and-later Intel processors.

	See Intel 64 Architecture SDM Vol. 3A "17.12.1 Invariant TSC".

	* Constant TSC

    The TSC increments at a constant rate, even CPU frequency get changed. But the TSC could be stopped when CPU run into
	deep C-state. Constant TSC is supported before Nehalem, and not as good as invariant TSC.

    * Variant TSC

    The first generation of TSC, the TSC increments could be impacted by CPU frequency changes.
	This is started from a very old processors (P4).


	Linux kernel defined a cpu feature flag CONSTANT_TSC for Constant TSC, and CONSTANT_TSC plus NONSTOP_TSC combinations
	for Invariant TSC.
	Please refer to [this kernel patch](https://github.com/torvalds/linux/commit/40fb17152c50a69dc304dd632131c2f41281ce44)
	for implementation.

	If CPU has no "Invariant TSC" feature, it might cause the TSC problems, when kernel enables P or C state: as known as
	turbo boost, speed-step, or CPU power management features.

	For example, if NONSTOP_TSC feature is not detected by Linux kernel, when CPU ran into deep C-state for power saving,
	[Intel idle driver](https://github.com/torvalds/linux/blame/master/drivers/idle/intel_idle.c)
	will try to mark TSC with unstable flag,

		if (((mwait_cstate + 1) > 2) &&
			!boot_cpu_has(X86_FEATURE_NONSTOP_TSC))
			mark_tsc_unstable("TSC halts in idle"
					" states deeper than C2");

	The ACPI CPU idle driver has the similar logic to check NONSTOP_TSC for deep C-state.

	Please use below command on Linux to check CPU capabilities,

		$ cat  /proc/cpuinfo | grep -E "constant_tsc|nonstop_tsc"


	Because no CPU feature flags could be used to ensure TSC reliability. The TSC sync test is the only way to test TSC
	reliability. However, some virtualization solution does provide good TSC sync mechanism. In order to handle some false
	positive test results, VMware create a new synthetic
	[TSC_RELIABLE feature bit](https://github.com/torvalds/linux/commit/b2bcc7b299f37037b4a78dc1538e5d6508ae8110)
	in Linux kernel to bypass TSC sync testing. This flag is also used by other kernel components to by pass TSC sync
	testing. Below command could be used to check this new synthetic CPU feature,

		$ cat  /proc/cpuinfo | grep "tsc_reliable"

	If we could get the feature bit set on CPU, we should be able to trust the TSC source on this platform. But keep in
	mind, software bugs in TSC handling still could cause the problems.


2. TSC sync behavior differences on Intel SMP system

	* No sync mechanism

    On most older SMP and early multi-core machines, TSC was not synchronized between processors. Thus if an application
	were to read the TSC on one processor, then was moved by the OS to another processor, then read TSC again, it might
	appear that "time went backwards". 

	Both "Variant TSC" and "Constant TSC" CPUs on SMP machine have this problem.

	* Sync among multiple CPU sockets on same main-board

	After CPU supports "Invariant TSC", most recent SMP system could make sure TSC got synced among multiple CPUs. At boot
	time, all CPUs connected with same RESET signal got reseted and TSCs are increased at same rate.

	* No sync mechanism cross multiple cabinets, blades, or main-boards.

	Depending on board and computer manufacturer design, different CPUs from different boards may connect to different
	clock signal, which has no guarantee of TSC sync.

	For example, on SGI UV systems, the TSC is not synchronized across blades. 
	[A patch provided by SGI](https://github.com/torvalds/linux/commit/14be1f7454ea96ee614467a49cf018a1a383b189) tries to
	disable TSC clock source for this kind of platform.

	Overall, even a CPU has "Invariant TSC" feature, but the SMP system still can not provide reliable TSC.
	For this reason, Linux kernel has to rely on some
	[boot time or runtime testing](https://github.com/torvalds/linux/commit/95492e4646e5de8b43d9a7908d6177fb737b61f0)
	instead of just detect CPU capabilities. The test sync test code used to have TSC value fix up code by calling write_tsc
	code. Actually, Intel CPU provides a MSR register which allows software change the TSC value. This is how write_tsc
	works. However, it is difficult to issue per CPU instructions over multiple CPUs to make TSC got a sync value. For this
	reason, Linux kernel just check the tsc sync and will not to write to tsc now.

	If TSC sync test passed during Linux kernel boot, following sysfs file would export tsc as current clock source,

		$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource
		tsc

	User application who relies on TSC usage may check this file to confirm whether TSC is reliable or not. However,
	per the root causes of TSC problems, kernel may not able to test out all of unreliable cases. It is still possible
	that TSC clock had the problem, and application developers need to handle all of them without kernel helps.

3. Non-intel x86 SMP platform

	Non-intel platform has different stories. Current Linux kernel treats all non-intel SMP system as non-sync TSC system.
	See unsynchronized_tsc code in [tsc.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/tsc.c).
	LKML also has the [AMD documents](https://lkml.org/lkml/2005/11/4/173).

4. Firmware problems

	As TSC value is writeable, firmware code could change TSC value that caused TSC sync issue in OS.
	There is [a LKML discussion](https://lwn.net/Articles/388286/) mentioned some BIOS SMI handler try to hide its execution
	by changing TSC value.


5. Hardware erratas

	TSC sync functionality was highly depends on board manufacturer design. For example, clock source reliability issues.
	I used to encountered a hardware errata caused by unreliable clock source. Due to the errata, Linux kernel TSC sync
	test code
	(check_tsc_sync_source in [tsc_sync.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/tsc_sync.c))
	reported error messages and disabled TSC as clock source.

	[Another LKML discussion](https://lwn.net/Articles/388188/) also mentioned that SMP TSC drift in the clock signals
	due to temperature problem. This finally could cause Linux detected the TSC wrap problems.

###Software TSC usage bugs

1. Overflow issues in TSC calculation

	The direct return value of rdtsc is CPU cycle, but latency or time requires a regular time unit: ns or us.

	In theory, 64bit TSC register is good enough for saving the CPU cycle, per Intel 64 Architecture SDM Vol. 3A 2-33,

	<pre> The time-stamp counter is a model-specific 64-bit counter that is reset to zero each
	time the processor is reset. If not reset, the counter will increment ~9.5 x 1016
	times per year when the processor is operating at a clock rate of 3GHz. At this
	clock frequency, it would take over 190 years for the counter to wrap around.</pre>

	The overflow problem here is in implementations of cycle_2_ns or cycle_2_us, which need multiply cycle with
	another big number, then this may cause the overflow problem.

	Per current Linux implementation, when the overflow bug may happen if,

	* The Linux OS has been running for more than 208 days
	* The Linux OS reboot does not cause TSC reset due to kexec feature
	* Some possible Hardware/Firmware bugs that cause no TSC reset during Linux OS reboot

	Linux kernel used to get suffered from the overflow bugs
	([patch for v3.2](https://github.com/torvalds/linux/commit/4cecf6d401a01d054afc1e5f605bcbfe553cb9b9)
	 [patch for v3.4](https://github.com/torvalds/linux/commit/9993bc635d01a6ee7f6b833b4ee65ce7c06350b1))
	when it try to use TSC to get a scheduler clock.

	In order to avoid overflow bugs, cycle_2_ns in Linux kernel becomes more complex than we referred before,

		 * ns = cycles * cyc2ns_scale / SC
		 *
		 * Although we may still have enough bits to store the value of ns,
		 * in some cases, we may not have enough bits to store cycles * cyc2ns_scale,
		 * leading to an incorrect result.
		 *
		 * To avoid this, we can decompose 'cycles' into quotient and remainder
		 * of division by SC.  Then,
		 *
		 * ns = (quot * SC + rem) * cyc2ns_scale / SC
		 *    = quot * cyc2ns_scale + (rem * cyc2ns_scale) / SC
		 *
		 *			- sqazi@google.com

	Unlike Linux kernel, some user applications uses below formula, which can cause the overflow if TSC cycles are more than
	2 hours!

		ns = cycles * 10^6 / cpu_khz

	Anyway, be careful for overflow issue when you use rdtsc value for a calculation.

2. Precision issues caused by GHZ/MHZ CPU frequency usage

	You may already notice that, Linux kernel use CPU KHZ instead of GHZ/MHZ in its implementation. Please read
	previous section about cycle_2_ns implementation.
	The major reason of using KHZ here is: better precision. Old kernel code used MHZ before, and
	[this patch](https://github.com/torvalds/linux/commit/dacb16b1a034fa7a0b868ee30758119fbfd90bc1) fixed the issue.

3. Precision issues caused by CPU out of order execution

	If the code you want to measure is a **very small** piece of code, our rdtsc function above might need to be
	re-implement by LFENCE or RDTSCP.

	See the description in Intel 64 Architecture SDM Vol. 2B,

	<pre> The RDTSC instruction is not a serializing instruction. It does not necessarily wait
	until all previous instructions have been executed before reading the counter. Similarly,
	subsequent instructions may begin execution before the read operation is
	performed. If software requires RDTSC to be executed only after all previous instructions
	have completed locally, it can either use RDTSCP (if the processor supports that
	instruction) or execute the sequence LFENCE;RDTSC.</pre>

	Linux kernel has an example to have the
	[rdtsc_ordered implementation](https://github.com/torvalds/linux/commit/03b9730b769fc4d87e40f6104f4c5b2e43889f19)


###TSC emulation on different hypervisors

Virtualization technology caused the lot of challenges for guest OS time keeping. This section just cover the cases
that host could detect the TSC clock source, and guest software may TSC sensitive and try to issue rdtsc instruction
to access TSC register while the task is running on a vCPU.

Per hypervisors differences, the rdtsc instruction could be executed with following ways,

1. Native - fast but potentially incorrect

	No emulation by hypervisor. The instruction is directly executed on physical CPUs.
	This mode has faster performance but may cause the TSC sync problems to TSC sensitive applications in guest OS.

	Especially, VM could be live migrated to different machine. It is not possible and reasonable to ensure TSC value
	got synced among different machines.

2. Emulated - correct but slow

	- Full virtualization

	Hypervisor will emulate TSC, then rdtsc is not directly executed on physical CPUs.
	This mode causes performance degrade for rdtsc instruction, but give the reliability for TSC sensitive application.

	- Para virtualization

	In order to optimize the rdtsc performance, some hypervisor provided PVRDTSCP which allows software in VM could be
	paravirtualized (modified) for better performance.

3. Hybrid - correct but potentially slow

	A hybrid algorithm to ensure correctness per following factors,

	- The requirement of correctness
	- Hardware TSC capabilities
	- Some special VM use case scenarios: VM is saved/restored/migrated

	When native run could get both good performance and correctness, it will be run natively without emulation.
	If hypervisor could not use native way, it will use full or para virtualization technology to make sure the correctness.

Below is detailed information about the TSC support on various hypervisors,

* VMware

ESX 4.x and 3.x does not make TSC sync between vCPUs.
But since ESX 5.x, the hypervisor always maintain the TSC got synced between vCPUs.
VMware uses the hybrid algorithm to make sure TSC got synced even if underlaying hardware does not support TSC sync.
For hardware with good TSC sync support, the rdtsc emulation could get good performance. But when hardware could not
give TSC sync support, TSC emulation would be slower.

In Linux guest, VMware creates a new synthetic TSC_RELIABLE feature bit to make sure it could by pass Linux TSC sync testing.
Linux [VMware cpu detect code] gives the good comments about TSC sync testing issues,

	/*
	 * VMware hypervisor takes care of exporting a reliable TSC to the guest.
	 * Still, due to timing difference when running on virtual cpus, the TSC can
	 * be marked as unstable in some cases. For example, the TSC sync check at
	 * bootup can fail due to a marginal offset between vcpus' TSCs (though the
	 * TSCs do not drift from each other).  Also, the ACPI PM timer clocksource
	 * is not suitable as a watchdog when running on a hypervisor because the
	 * kernel may miss a wrap of the counter if the vcpu is descheduled for a
	 * long time. To skip these checks at runtime we set these capability bits,
	 * so that the kernel could just trust the hypervisor with providing a
	 * reliable virtual TSC that is suitable for timekeeping.
	 */
	static void vmware_set_cpu_features(struct cpuinfo_x86 *c)
	{
		set_cpu_cap(c, X86_FEATURE_CONSTANT_TSC);
		set_cpu_cap(c, X86_FEATURE_TSC_RELIABLE);
	}

The [patch](https://github.com/torvalds/linux/commit/eca0cd028bdf0f6aaceb0d023e9c7501079a7dda) got merged in Linux 2.6.29.

VMware also provides
[Timekeeping in VMware Virtual Machines](http://www.vmware.com/files/pdf/Timekeeping-In-VirtualMachines.pdf) to discuss
TSC emulation issues. Please refer to this document for detailed information.


* Hyper-V

	Hyper-V does not provide TSC emulation. For this reason, TSC on hyper-V is not reliable. But the problem is, hyper-V
	Linux CPU driver never reported the problem, that means the TSC clock source is still could be used if it happed to
	pass Linux kernel TSC sync test.
	Just 20 Days ago, a Linux kernel
	[4.3-rc1 patch](https://github.com/torvalds/linux/commit/88c9281a9fba67636ab26c1fd6afbc78a632374f)
	had disabled the TSC clock source on Hyper-V Linux guest.

* KVM

	TBD

* Xen

	Prior to Xen 4.0, it only support native mode.
	Xen 4.0 provides tsc_mode parameter, which allows administrators switch between 4 modes per their requirements.
	By default Xen 4.0 use the hybrid mode. [This Xen document](http://xenbits.xen.org/docs/4.2-testing/misc/tscmode.txt)
	gives very detailed discussion about TSC emulation.

##Conclusion

1. If possible, avoid to use rdtsc in user applications.

   Not all of hardware, hypervisors are TSC safe, which means TSC may behave incorrectly.

   TSC usage will cause software porting bugs cross various x86 platforms or different hypervisors.
   Leverage syscall or vsyscall will make software portable, especially for Virtualization environment.

2. If you have to use it, you must understand the risks from hardware, OS kernel, hypervisors.

   Use it for debugging, but never use rdtsc in functional area.
   
   As we mentioned above, Linux kernel also had hard time to handle it until today. If possible, learn from Linux code first.
   Perf measurement and debug facility might be only usable cases, but be prepare for handling various conner cases
   and software porting problems.
 
   Please make sure you can write a "TSC-resilient" application, which make sure your application still can behave
   correctly when TSC value is wrong.