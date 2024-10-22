# GHNAVS gem5 simulator

GHNAVS stands for Gem5 - HBM - NoC - ARM - V1 - Simulator

This is a patch for gem5, to enhance support for ARM64 CPUs and ARM-CMN Interconnects modeling.

## I. Features 

1. Implementation for multi-layered NoC, where each traffic class (VNET) uses a separate NoC. This is a more accurate modeling of ARM CMN Interconnects[7], in relation to the existing gem5 NoC models (`garnet2.0` [1]).
    * The number of NoC layers be adjusted with a single parameter (`--noc_layers`) from the command-line

2. Changes in the Ruby CPU Sequencer [2, 3], in order to avoid Sequencer port blocking caused by aliased requests. In the default implementation of Ruby CPU Sequencer subsequent requests (read or write), to the same Cache line are called aliased requests, and cause the CPU master port to block, even for requests targeting a different cacheline. This unrealistic modeling results in very poor performance (due to low number of outstanding requests), for workloads with sequential access to memory (e.g: STREAM [4]).
    * This new feature (called `sequencer_port_block_bypass`) can be turned on/off using a single configuration parameter.

3. Processor models: 
  - `ARM Cortex-A76 CPU` and 
  - `R-CPU`, which models an ARM-Neoverse-V1 CPU core with SVE support

4. NoC Topology: 16-cores, 16 SLC slices, 8 HBM2 controllers 4x4 Mesh Topology

5. HBM2 memory model 

6. Very detailed NoC statistics

7. MOESI_CMP_directory CC protocol

## II. Instructions

### A. How to build

#### Software Requirements

- g++ version 7 
- scons version 3.01
- python version 2.7

Additionally you are advised to install `pydot` and `graphviz` related libraries, in order to visualize the generated Systems, and cross-check that they are as expected. (When those libraries are installed gem5 will produce `config.dot.pdf` and `config.system.ruby.dot.pdf` files, which contain a high level, visual representation of the components of the simulated system).

One can build an LXC `debian buster` container, which can have all these requirements.

The provided patch is based on `public-gem5` source code [6], commit: `904784fb1e15f0c090fb1f1e5c5057e74b0b4ea8`.
In order to apply the patch use the following:


```
    git clone https://gem5.googlesource.com/public/gem5
    cd gem5
    git checkout 904784fb1e15f0c090fb1f1e5c5057e74b0b4ea8
    git apply *.patch 
```

Example build command:

```
   CXX=g++-7 scons NUMBER_BITS_PER_SET=128 PROTOCOL=MOESI_CMP_directory build/ARM/gem5.opt -j7 --force-lto
```

### B. Important configuration parameters

The following parameters, can be adjusted to more accurate values, depending on the system you want to setup.

| Param | File | Value | Description |
| --- | --- | --- | --- |
| max_outstanding_requests | src/mem/ruby/system/Sequencer.py | 128 | The number of outstanding requests for Ruby CPU Sequencer |
| sequencer_port_block_bypass | src/mem/ruby/system/Sequencer.py | True | Use True to enable FORTH Sequencer (False for the default Ruby Sequencer) |
| --sys-clock | command-line | 2GHz | System Clock |
| --ruby-clock | command-line | 2GHz | Ruby Subsystem Clock |
| --cpu-clock | command-line | 2GHz | CPU clock |
| --topology | command-line | Mesh_EPI_quadrant_p1 | Mesh_XY with multiple NoC layers support |
| --mesh-rows | command-line | 4 | 4x4 Mesh, (set to 4 for 4x4 Mesh) |
| --link-width-bits | command-line | 576 | This covers 8 bytes for control and 64 bytes for data (64 bit + 512 bits) |
| --noc_layers | command-line | 3 | The number of layers is dependent on the number of VNETS of the CC protocol that is used. Use 3 for MOESI_CMP_directory protocol. Use 6 if you want to enable the multi-VNETs NoC feature |
| sve_vl_se | src/arch/arm/ArmISA.py | 1/2/4/8/16 | SVE Length in Quadwords (quadword = 128 bits). **This only works for SE mode** |
| sve_vl | arch/arm/ArmSystem.py | 1/2/4/8/16 | SVE Length in Quadwords (quadword = 128 bits). **This only works for FS mode** |
| --num-dirs | command-line | 4,8,... | **On Ruby Systems, the number of memory channels is directly related to the number of  Cache directories. Note that `mem-channels` parameter is ignored when modeling Ruby Systems** |
| --mem-channels | command-line | 1,2,4,8,... | **On Classic Memory Systems the number of memory channels is adjusted by `mem-channels`** |
| --ports | command-line | 4 | For Ruby Systems: CC transitions per cycle |

### C. How to run

#### 1. Syscall emulation (SE) case (`se.py`)

An example command for SE mode is the following:

```
    ./build/ARM/gem5.opt \
    --outdir=m5out/outdir_1 \
    configs/example/se.py \
    --cpu-type=R_CPU \
    --arm-iset=aarch64 \
    --num-cpus=4 \
    --num-dirs=4 \
    --caches \
    --l2cache \
    --num-l2caches=4 \
    --l1i_size=64kB \
    --l1d_size=64kB \
    --l2_size=1MB \
    --l1i_assoc=4 \
    --l1d_assoc=4 \
    --l2_assoc=8 \
    --mem-type=DDR4_2400_8x8 \
    --mem-size=2GB \
    --sys-clock=2GHz \
    --cpu-clock=2GHz \
    --ruby-clock=2GHz \
    --ruby \
    --topology=Mesh_EPI \
    --mesh-rows=2 \
    --network=garnet2.0 \
    --link-width-bits=576 \
    --noc_layers=3 \
    --vcs-per-vnet=4 \
    -c sve_stream_copy_se.exe
```
`Note:` One can use `-c $executable1;$executable2;$executable3;$executable4` to launch the same or different applications to each core, since mutli-thread support is limited in SE mode.

#### 2. FullSystem (FS) mode (`fs.py`)

For Full System mode we use `fs.py` simulation script, as it supports the gem5 Ruby System [2], and therefore NoC modeling.

* Getting a checkpoint: We always get a checkpoint using a classic system setup. The number of CPU cores, SVE lengths, and memory size are important as changing those requires a new checkpoint. Additionally mounting or adding files to the image file requires a new checkpoint to be taken.

```
./build/ARM/gem5.opt \
--outdir=m5out/checkpoint_outdir \
configs/example/fs.py \
--cpu-type=AtomicSimpleCPU \
--cpu-clock=2GHz \
--num-cpus=16 \
--kernel=vmlinux_bsc \
--disk=aarch64-ubuntu-armcl-sve.img \
--machine-type=VExpress_GEM5_V1 \
--mem-type=DDR4_2400_8x8 \
--script=configs/boot/hack_back_ckpt.rcS \
--mem-size=2GB
```

* Restoring from a checkpoint: In order to restore from a checkpoint, the checkpoint directory (named `cpt.*`) should be copied (or sym-linked), inside the outdir of the new simulation (`outdir_2` in the following examples).
* If you don't want to restore from a checkpoint, remove `-r 1` from the command line.

#### 2-a) Simple system with a 2x2 Mesh Noc

![8 core - Ruby NoC System - MOESI_CMP_directory](images/simple_noc.png?raw=true "2x2 Mesh NoC")

#### 2-b) 4x4 Mesh: 16 cores - 16 SLCs - 8 HBM2 controllers (Topology: Mesh_EPI_quadrant_p1)

```
CXX=g++-7 scons NUMBER_BITS_PER_SET=128 PROTOCOL=MOESI_CMP_directory build/ARM/gem5.opt -j7 --force-lto

./build/ARM/gem5.opt \
--listener-mode=off  \
 --outdir=m5out/outdir_2 \
 configs/example/fs.py \
 -r 1 \
 --cpu-type=R_CPU \
 --restore-with-cpu=R_CPU \
 --arm-iset=aarch64 \
 --kernel=vmlinux_bsc \
 --disk=aarch64-ubuntu-armcl-sve.img \
 --machine-type=VExpress_GEM5_V1 \
 --script=configs/boot/bootscript1.rcS \
 --num-cpus=16 \
 --num-dirs=8 \
 --num-l2caches=16 \
 --caches \
 --l2cache \
 --l1i_size=64kB \
 --l1d_size=1MB \
 --l2_size=2MB \
 --l1i_assoc=4 \
 --l1d_assoc=4 \
 --l2_assoc=8 \
 --mem-type=HBM2_2000_4H_1x128 \
 --mem-size=2GB \
 --sys-clock=2GHz \
 --cpu-clock=2GHz \
 --ruby-clock=2GHz \
 --ruby \
 --topology=Mesh_EPI_quadrant_p1 \
 --mesh-rows=4 \
 --network=garnet2.0 \
 --link-width-bits=576 \
 --noc_layers=3 \
 --vcs-per-vnet=4
 ```

![EPI Quadrant Topology](images/Topology_T1_gem5opensource.png?raw=true "4x4 Mesh Topology")

## III. Advanced features

### A. Setup with SLC approximation

MOESI_CMP_directory only supports L1 and L2. However we can approximate a 3-level cache hierarchy using the following:
The idea is to use the L2 cache controllers to approximate the SLC cache slices. 
L1D size is increased to match the size of L1+L2 of the modeled platform, and L2 size is increased to match the size of the platform SLC.
Also the cache latencies are adjusted accordingly.

The provided patch already adjusts L1/L2 cache latencies of the model. Then, you can adjust the `L1D` size, `L2` size (which acts as SLC), as well as the number of L2 Controllers (`--num-l2caches`), approximating the number of SLC slices.


### B. Mutli-VNETs feature (Double NoC bandwidth)

This release can support having multiple VNETs per Request/Response (This means that each VNET can have 2 physical links).  More specifically one can initiate a gem5 system with a total of `6` NoC layers, instead of `3`, while using `MOESI_CMP_directory`. Use `--noc_layers=6`, and make sure that `use_offset_vnets = True` in file: `configs/network/Network.py`.

### C. Detailed NoC Latencies (and CC message types)

One can print detailed per VNET and per Source to Destination NoC Queue and Network latencies. To enable this feature, use `--debug-flags=RubyNetConnections`. When using this debug flag, the output shown, will contain two types of messages. 

The first one regards the NoC latencies. For example:

```
    7207675622500: system.ruby.network: vnet:2, [L1Cache_Controller 3]->[L2Cache_Controller 5],
    NI[3->21], Rtr[46->37], hops[3],
    queue_AvgMinMaxMed[8.06/8/53/8.00],
    net_AvgMinMaxMed[9.86/9/91/9.00],
    flits:29393
```
The message is split into the following fields:

1. gem5 simulation current tick
1. VNET number
1. Source Controller -> Destination Controller (**Reminder: L1 represents L1+L2 and L2 represents SLC**)
1. Source NI -> Destination NI
1. Source Router -> Destination Router
1. Number of hops
1. Queueing latencies (Avg, Min, Max, Median)
1. Network latencies (Avg, Min, Max, Median)
1. Number of packets (flits) sent from source to destination during the last ROI

The second type of messages, describes what type of CC messages, each Source Controller sends to each Destination Controller per VNET and ROI. An example printout is the following:

```
7207675622500: system.ruby.network: vnet:1, [L2Cache_Controller 2]->[Directory_Controller 0], REQ: [GETX : 19691][GETS : 39488][PUTX : 19723][WRITEBACK_DIRTY_DATA : 19723]
7207675622500: system.ruby.network: vnet:2, [L1Cache_Controller 3]->[L2Cache_Controller 0], RESP: [UNBLOCK : 19713][UNBLOCK_EXCLUSIVE : 9781]
```
Those types of messages contain the following fields:

1. gem5 simulation current tick
1. VNET number
1. Source Controller -> Destination Controller (**Reminder: L1 represents L1+L2 and L2 represents SLC**)
1. REQ / RESP (Request, Response Type)
1. A sequence of: [CC Message type : Number of messages during last ROI]

## IV. References

[1] [http://old.gem5.org/Garnet2.0.html](http://old.gem5.org/Garnet2.0.html)

[2] [http://old.gem5.org/Ruby.html](http://old.gem5.org/Ruby.html)

[3] [http://old.gem5.org/Coherence-Protocol-Independent_Memory_Components.html](http://old.gem5.org/Coherence-Protocol-Independent_Memory_Components.html)

[4] [https://www.cs.virginia.edu/stream/](https://www.cs.virginia.edu/stream/)

[5] [http://old.gem5.org/Running_gem5.html](http://old.gem5.org/Running_gem5.html)

[6] [https://gem5.googlesource.com/public/gem5](https://gem5.googlesource.com/public/gem5)

[7] [Arm Neoverse CMN‑650 Coherent Mesh Network TRM](https://developer.arm.com/documentation/101481/0200/What-is-CMN-650-/Product-documentation-and-design-flow)

## V. Acknowledgements

We thankfully acknowledge support for this research from the European High Performance Computing Joint Undertaking (EuroHPC JU) under Framework Partnership Agreement No 800928 (European Processor Initiative) and Specific Grant Agreement No 101036168 (EPI-SGA2). The EuroHPC JU receives support from the European Union’s Horizon 2020 research and innovation programme and from Croatia, France, Germany, Greece, Italy, Netherlands, Portugal, Spain, Sweden, and Switzerland. National contributions from the involved state members (including the Greek General Secretariat for Research and Innovation) match the EuroHPC funding.
