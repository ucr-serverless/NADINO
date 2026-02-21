## About

**NADINO** is a research prototype for high-performance, DPU-accelerated serverless networking. NADINO consists of two main components:
* **[NADINO Ingress](https://github.com/ucr-serverless/nadino-ingress.git):** an NGINX-based ingress that offloads protocol conversion and communication to RDMA and F-Stack.
* **[NADINO Network Engine](https://github.com/ucr-serverless/nadino-network-engine.git):** a DPU-enabled network engine that orchestrates RDMA flows and enables function chaining with low CPU overhead. Supports two deployment variants:
  * **DNE (DPU Network Engine):** gateway runs on a BlueField-2 DPU.
  * **CNE (CPU Network Engine):** gateway runs on the host CPU.

---

## Table of Contents

* [Testbed](#testbed)
* [Installation](#installation)
  * [Ingress](#nadino-ingress)
  * [Network Engine](#network-engine)
* [Workload Configuration](#workload-configuration)
* [Sample Experiments](#sample-experiments)
* [License](#license)

---

## Testbed

- **Tested OS**: Ubuntu 22.04 with kernel 5.15.
- **Reference topology**: You will need at least four nodes in your cluster. Two nodes are used as worker nodes to deploy user functions. One node is used to deploy NADINO ingress. The remaining node is used for load generation. A sample topology is shown below:

    <img src="./docs/ref_topo.png" alt="ref_topo" style="width:50%; height:auto;">

- **NIC/DPU Requirement**:
    - NADINO ingress requires **two** NICs: One [DPDK-compatible NIC](https://core.dpdk.org/supported/nics/) for F-stack and one RDMA (Mellanox/NVIDIA ConnectX) NIC.
    - Worker nodes require a NVIDIA BlueField DPU to deploy NADINO DNE. CNE runs on the host CPU without a DPU.

> We recommend using [`r7525`](https://docs.cloudlab.us/hardware.html) nodes on CloudLab with our customized [network profile](https://www.cloudlab.us/p/KKProjects/dpu-same-lan).

---

## Installation

Clone NADINO and initialize submodules:

   ```bash
   git clone --recursive https://github.com/ucr-serverless/NADINO.git
   cd NADINO
   git submodule update --init --recursive
   ```

### NADINO Ingress

> Full installation guide: [nadino-ingress/README.md](./nadino-ingress/README.md)

Key steps:

1. Install build dependencies (`flex`, `bison`, `libssl-dev`, `libelf-dev`, `libnuma-dev`, `libconfig-dev`, `uuid-dev`, `libpcre3-dev`, `libglib2.0-dev`, etc.) and DOCA 2.10.0.
2. Build DPDK 21.11 and F-stack (install DOCA before this step so the mlx5 PMD is compiled in).
3. Configure hugepages and NIC binding — PCIe `pci_whitelist` for Mellanox; `igb_uio` for Intel.
4. Build RDMA lib, DOCA lib, and NADINO Ingress:

    ```bash
    cd ~/NADINO/nadino-ingress/

    # Build RDMA lib
    cd RDMA_lib && make && cd ..

    # Build DOCA lib
    cd DOCA_lib && meson /tmp/doca_lib && ninja -C /tmp/doca_lib && cd ..

    # Build and install NADINO Ingress
    FF_PATH=~/nadino-ingress/f-stack \
    PKG_CONFIG_PATH=/usr/lib64/pkgconfig:/usr/local/lib64/pkgconfig:/usr/lib/pkgconfig \
        ./configure --prefix=/usr/local/nginx_fstack --with-ff_module
    python ./scripts/patch_make.py   # patches objs/Makefile for pdi_rdma
    FF_PATH=~/nadino-ingress/f-stack \
    PKG_CONFIG_PATH=/usr/lib64/pkgconfig:/usr/local/lib64/pkgconfig:/usr/lib/pkgconfig \
        make -j
    sudo make install
    ```

5. Configure `conf/f-stack.conf` (DPDK port and hugepage settings), `conf/nginx.conf` (worker count, location blocks), and `conf/rdma.cfg` (RDMA device, backend IP/port, GID index — read at runtime, no recompile needed). Run `sudo make install` after editing config files.

6. Run NADINO Ingress:

    ```bash
    sudo /usr/local/nginx_fstack/sbin/nginx -g "daemon off;"
    ```

---

### Network Engine

> Full installation guide: [nadino-network-engine/README.md](./nadino-network-engine/README.md)

Key steps:

1. Install DOCA 2.10.0 on each host node. For DPU setup, see the [BlueField2 DPU Setup Guide](./nadino-network-engine/docs/BlueField2-DPU-Setup-Guide.md).

2. Run environment setup scripts to install libbpf, DPDK RTE libraries, and configure hugepages:

    ```bash
    cd ~/NADINO/nadino-network-engine
    bash sigcomm-experiment/env-setup/001-env_setup_master.sh
    bash sigcomm-experiment/env-setup/002-env_setup_master.sh
    ```

3. Build RDMA lib, DOCA lib (DNE only), and Network Engine:

    ```bash
    cd RDMA_lib && meson setup build --reconfigure && ninja -C build/ -v && cd ..
    cd DOCA_lib && meson /tmp/doca_lib && ninja -C /tmp/doca_lib && cd ..  # DNE only
    meson setup build && ninja -C build/ -v
    ```

All components are launched via `run.sh` (run as root, in order):

| Component | Command | Notes |
|-----------|---------|-------|
| Shared memory manager | `sudo ./run.sh shm_mgr <cfg>` | Start first on each node |
| DPU gateway | `sudo ./run.sh gateway <cfg>` | DNE: runs on BlueField DPU |
| CPU gateway | `sudo ./run.sh cpu_gateway <cfg>` | CNE: runs on host CPU |
| Network function | `sudo ./run.sh <service_name> <nf_id>` | One process per function |
| Sockmap manager | `sudo ./run.sh sockmap_manager` | DNE only |

---

## Workload Configuration

* Update config files in `nadino-network-engine/cfg/` directory (see [CONFIG.md](docs/CONFIG.md)).

> The configuration file defines the mapping of functions, routes, nodes, tenants, memory, and RDMA settings needed to deploy and run NADINO experiments. It defines:
>
> * **Functions:** their identities, names, placement, threading, and workload parameters.
> * **Call graphs:** the execution paths requests follow across functions.
> * **Nodes:** the worker nodes and DPUs, along with IPs, hostnames, RDMA devices, and DOCA communication channel settings.
> * **Tenants:** isolation and fairness settings via tenant IDs, weights, and permitted routes.
> * **Memory manager settings:** memory pool sizes, placement, and associated device bindings.
> * **RDMA settings:** whether to use RDMA or TCP, choice of one-sided vs. two-sided RDMA, queue sizing, and experiment knobs.

To auto-detect RDMA parameters on a CloudLab node:

```bash
python RDMA_lib/scripts/get_cloudlab_node_settings.py
```

---

## Sample Experiments

### Running with a Dummy Function Chain
See [Dummy Function Chain Setup](docs/DUMMY.md) for details.

### Running with Online Boutique Workload
See [Online Boutique Workload](docs/ONLINE_BOUTIQUE.md) for details.

---

## License

* Cluster Ingress: BSD 3-Clause (with Apache-2.0 modifications)
* NADINO Network Engine: Apache License 2.0

See individual `LICENSE` files for details.
