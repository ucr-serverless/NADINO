# Configuring Workload

> Sample configurations are available under `nadino-network-engine/cfg/`. The configuration file specifies all necessary parameters for deploying and running **NADINO Network Engine** and **functions**. It defines:

* **Functions:** their identities, names, placement, threading, and workload parameters.
* **Call graphs:** the execution paths requests follow across functions.
* **Nodes:** the worker nodes and DPUs, along with IPs, hostnames, RDMA devices, and DOCA communication channel settings.
* **Tenants:** isolation and fairness settings via tenant IDs, weights, and permitted routes.
* **Memory manager settings:** memory pool sizes, placement, and associated device bindings.
* **RDMA settings:** whether to use RDMA or TCP, choice of one-sided vs. two-sided RDMA, queue sizing, and experiment knobs.

---

# How to use it (at a glance)

1. **Adjust hostnames/IPs/devices** in the `nodes` block to match your machines (`hostname`, `ip_address`, `rdma_device`, PCI IDs like `0000:03:00.0`).
2. (Optional) **Tune functions** in `nfs` (thread counts, synthetic load via `params`).
3. (Optional) **Pick a route** in `routes` and ensure its `hops` refer to valid function `id`s.
4. **Memory manager**: confirm `memory_manager.ip/port` and `local_mempool_size`.
5. **RDMA settings**: set `use_rdma` (0=TCP, 1=RDMA) and `use_one_side` (0=two-sided, 1=one-sided).

---

# Key parameters

## 1) `nfs`: functions and placement

Each entry describes a function instance:

* `id`: Integer handle used by routes (must be unique).
* `name`: Human-readable service name (e.g., `"frontendservice"`).
* `tenant_id`: Tenant this function belongs to (here, single-tenant `0`).
* `n_threads`: Worker threads inside this function.
* `params`: Synthetic load controls:

  * `memory_mb`: Size of memory to touch/allocate for the function.
  * `sleep_ns`: Per-request sleep (think: artificial latency).
  * `compute`: A unitless knob for CPU work (implementation-specific).
* `node`: Which **node index** (from the `nodes` array) to place this function on.
* `mode`: **Function activity mode** (from your comment in config):

  * `0` = passive (receives requests)
  * `1` = active (generates requests)

---

## 2) `routes`: request paths (function chains)

Each route is a sequence of **function ids**:

* `hops`: e.g., `[1, 2, 1, 2, 1]` means the request visits Fn#1 → Fn#2 → Fn#1 → Fn#2 → Fn#1.
* Define as many routes as you like; your traffic generator can pick among them.

> Tip: Keep routes consistent with `nfs.id` values; if you add or remove functions, update `hops`.

---

## 3) `nodes`: worker + DPU mapping and NIC/RDMA info

Defines each worker node (host) and its attached DPU:

* `id`: Node index used by `nfs[].node`.
* `hostname`: Host machine’s name (must match actual host or your inventory).
* `dpu_hostname`: The DPU’s hostname paired with this host.
* `ip_address`: Host IP.
* `dpu_ip_addr`: DPU IP.
* `port`: Service/control port used by network engine on that node.
* `rdma_device`: RDMA device name (e.g., `mlx5_2`) as seen by verbs/OFED.
* `comch_*_device`: PCI addresses for communication channels (control/data paths).
  These are environment-specific; verify via `lspci`/`ibv_devinfo`.

  * `comch_server_device`
  * `comch_client_device`
  * `comch_client_rep_device` (often the representor used by the client side)
* `sgid_idx`: SGID index to use (check with `show_gids`).
* `mode`: **Deployment mode selector.**
  You have `mode = 4` here. (If you maintain multiple modes, document meanings alongside your code/scripts; keep values consistent across nodes.)
* `receive_req`: `0` = not connected to ingress, `1` = will receive from ingress.
* `mm_ip` / `mm_port`: Where the memory manager listens for this node.

> Practical checks:
>
> * `rdma_device` must exist on the host (`ibv_devinfo -l`).
> * PCI IDs (`0000:xx:yy.z`) must match your NIC functions/representors.
> * `sgid_idx` must be valid for the chosen device/port (`show_gids`).

| Field        | Description                                                         |
| ------------ | ------------------------------------------------------------------- |
| `device_idx` | Index of the RDMA device                                            |
| `sgid_idx`   | Index of the SGID to be used                                        |
| `ib_port`    | RDMA device’s InfiniBand port index                                 |
| `qp_num`     | Total number of queue pairs to initialize (must match across nodes) |

⚠️ **Important:** The `qp_num` value must be consistent across **all nodes**.

To automatically detect `device_idx`, `sgid_idx`, and `ib_port` on CloudLab machines, run:

```bash
python RDMA_lib/scripts/get_cloudlab_node_settings.py
```

This script invokes `ibv_devinfo -v`, `lspci | grep 'Mellanox'`, and `show_gids` under the hood, and parses their output.

> ✅ Ensure the **OFED driver** is installed before running the script. Only **ConnectX-4 and newer** RDMA devices are supported.

---

## 4) `tenants`: multi-tenancy weights

* `id`: Tenant identifier.
* `weight`: Relative share for scheduling/fairness (higher → more share).
* `routes`: Which route IDs this tenant can use.

---

## 5) `memory_manager`: shared-memory pool and placement

* `local_mempool_size`: Elements in the local mempool.
  Increase if you see allocation/init failures; tune with your hugepage setup.
* `port`: Memory manager’s port.
* `ip`: Memory manager’s IP (the comment above suggests this also flags “remote” vs “same host” placement).
* `device`: RDMA device (e.g., `mlx5_0`) used by the memory manager.
* `is_remote_memory`: `1` if network engine is on DPU; `0` if same host.

---

## 6) `rdma_settings`: transport & experiment knobs

* `use_rdma`: `1` = use RDMA; `0` = use TCP sockets.
* `use_one_side`: `1` = one-sided; `0` = two-sided RDMA (only matters if `use_rdma=1`).
* `n_init_task`, `n_init_recv_req`: Initial sizes for internal queues/pools.
* `tenant_expt`: Enable tenant-specific experiment behavior (implementation-specific flag).
* `msg_sz`: Message size (bytes) for synthetic traffic.
* `n_msg`: Number of messages to send (synthetic runs).
* `ngx_ip`: IP of the ingress (if integrating).
* `ngx_id`: Worker ID for ingress (currently supports a single worker: `0`).
* `json_path`: Path to a JSON config for multi-tenancy experiments.
* `is_dummy_nf`: `1` = run dummy functions (synthetic workload); `0` = real functions.

> Common pitfalls:
>
> * Setting `use_rdma=1` without correct RDMA devices/SGIDs/OFED → init fails.
> * One-sided RDMA requires compatible code paths; if unsure, use two-sided (`use_one_side=0`).

If you are using **two-sided RDMA primitives** in Network Engine, most RDMA settings can remain unchanged, unless you [modify your huge page configuration](install-dependencies.md).

* **`local_mempool_size`**: Number of elements in the local mempool.
  If you adjust DPDK huge page allocations, update this value until `shm_mgr` initializes successfully.

The following options are provided **for reference only** and usually do not need modification:

* **`use_rdma`**
  Whether network engine should use RDMA (`1`) or TCP sockets (`0`).
  If set to `0`, the other RDMA-related options are ignored.

* **`use_one_side`**
  Selects between one-sided (`1`) and two-sided (`0`) RDMA primitives.

* **`mr_per_qp`**
  Number of memory regions per queue pair (valid only if `use_one_side=1`).
  For two-sided RDMA (`use_one_side=0`), leave this as `0`.

* **`init_cqe_num`**
  Initial size of the RDMA completion queue.
  The program attempts to allocate this many entries, but the actual number may be smaller.

* **`max_send_wr`**
  Upper bound on the send queue size for RDMA.
  The actual limit may be smaller than the value specified.

---

# Minimal customization checklist

1. **Map your machines**

   * `nodes[0]` ↔ worker/DPU#1, `nodes[1]` ↔ worker/DPU#2
   * Update `hostname`, `dpu_hostname`, `ip_address`, `dpu_ip_addr`, `mm_ip/mm_port`.

2. **Verify hardware identifiers**

   * `rdma_device` (`ibv_devinfo -l`)
   * `sgid_idx` (`show_gids`)
   * `comch_*_device` (via `lspci`, confirm representors)

3. **Pick transport**

   * **Start with TCP:** `use_rdma=0` to validate logic.
   * **Then RDMA:** set `use_rdma=1`; choose two-sided first (`use_one_side=0`).

4. **Tune function workload**

   * `n_threads`, `params.memory_mb/sleep_ns/compute` based on your experiment.

5. **Routes/tenants**

   * Ensure `routes[].hops` reference valid `nfs[].id`.
   * Tenant weights reflect your fairness experiment.

---

# Quick sanity commands

On each host (before running):

```bash
# RDMA device inventory
ibv_devinfo -l
show_gids | sed -n '1,100p'
lspci | grep -i mellanox

# Hugepages status (if applicable)
grep -H . /sys/kernel/mm/hugepages/hugepages-*/nr_hugepages
```

If something fails to initialize (e.g., completion queues, mempool), lower `local_mempool_size` or ensure hugepages/OFED are configured.
