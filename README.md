# CMPE 283 – Assignment 2: KVM Exit Instrumentation

## Questions – Provide answers in a README.md in the root of your repo.

### 1. For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched. (You may skip this question if you are doing the lab by yourself).

This assignment was completed individually by **Tamizhselvan Manivannan**.  
I completed all parts of the lab on my own — setting up the environment on Google Cloud Platform, building and modifying the Linux kernel, enabling KVM support, and running the inner virtual machine to collect and analyze the exit statistics. I also handled troubleshooting for kernel module and networking issues to get the nested virtualization working correctly.

---

### 2. Describe in detail the steps you used to complete the assignment. Consider your reader to be someone skilled in software development but otherwise unfamiliar with the assignment. Good answers to this question will be recipes that someone can follow to reproduce your development steps.
**Note:** I may decide to follow these instructions for random assignments, so you should make sure they are accurate.

1. **Environment Setup:**  
   I created an outer virtual machine on Google Cloud Platform (GCP) using the N2 series (Intel Cascade Lake) and enabled nested virtualization. I installed all the required development and virtualization packages (`build-essential`, `libvirt`, `qemu`, `virt-manager`, etc.). I verified hardware virtualization support using `kvm-ok` and confirmed that `/dev/kvm` existed.

2. **Kernel Preparation and Build:**  
   I forked the official Linux repository from GitHub (https://github.com/torvalds/linux) into my own account and cloned it inside the outer VM. I configured the kernel using `make defconfig` and tagged it as `-cmpe283-a2` for identification. I performed an initial build and install to confirm that the environment could compile and boot a custom kernel.

3. **KVM Instrumentation:**  
   I located the Intel VM-exit handler function `__vmx_handle_exit()` in the file `arch/x86/kvm/vmx/vmx.c`.  
   At the top of the file, I added global counters and a helper function named `cmpe283_maybe_dump_exits()` that prints the total and per-type exit counts every 10,000 VM exits. Inside the exit handler, right after the `exit_reason` variable is set, I incremented both the per-type and total counters and called the helper function.  
   I rebuilt the kernel with these changes, installed it, and rebooted into the new version.

4. **Kernel Configuration Updates:**  
   During configuration, I enabled the KVM and bridge networking options in `make menuconfig`:
   - `Kernel-based Virtual Machine (KVM) support`
   - `KVM for Intel processors support`
   - `802.1d Ethernet Bridging`  
   This allowed hardware acceleration and network bridge creation.

5. **Testing the Modified Kernel:**  
   After rebooting, I confirmed that the kernel was running (`uname -a` showed `-cmpe283-a2`) and that `/dev/kvm` existed. I used libvirt to create a default NAT network and launched an inner Ubuntu VM using `virt-install`. With the inner VM running, I opened another terminal on the outer VM and monitored `dmesg` for my `CMPE283` log messages.

6. **Verification:**  
   The kernel printed the total and per-exit-type counts every 10,000 exits. The log showed exit types such as `EPT_MISCONFIG`, `IO_INSTRUCTION`, `HLT`, and `CPUID` with increasing counts. This verified that my modifications were working as intended.

---

### 3. Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there more exits performed during certain VM operations? Approximately how many exits does a full VM boot entail?

From observation, the number of exits increases rapidly during the inner VM’s boot process and whenever heavy I/O operations occur, such as system startup or package updates. Once the guest becomes idle, the exit rate slows and stabilizes, mostly due to periodic interrupts and `HLT` instructions.  
A full inner-VM boot on my setup generated around **150,000–160,000 total VM exits**. After the guest reached the login prompt, exits increased more slowly.

---

### 4. Of the exit types, which are the most frequent? Least?

The **most frequent** exits observed were:
- `EPT_MISCONFIG` – about 95,000 occurrences  
- `IO_INSTRUCTION` – around 30,000–40,000 occurrences  
- `HLT` – about 4,000–5,000 occurrences  

These are expected since memory translations and I/O operations happen continuously during boot.  

The **least frequent** exits were:
- `EXCEPTION_NMI` – only about 15 occurrences  
- `exit 31` and `exit 54` – only one or two each  

This matches normal system behavior where interrupts and certain rare instructions occur infrequently compared to memory and I/O exits.

---

**Result Summary:**  
The modified kernel successfully counted and reported VM exits every 10,000 total exits, with correct per-type breakdowns. The instrumentation worked as intended and the results provided useful insight into which virtualization events occur most frequently during VM execution.
