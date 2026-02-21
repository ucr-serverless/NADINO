## Running with a Dummy Function Chain

This example demonstrates a minimal setup where two functions are chained linearly:

```
client → function-1 → function-2 → client
```

The deployment uses **two worker nodes**, each running a memory manager, gateway, and one function instance.

For full details see the [nadino-network-engine Quick Test](../nadino-network-engine/README.md#quick-test--two-node-dummy-function-chain).

---

### Worker Node 1

```bash
cd nadino-network-engine

# Start the shared memory manager
sudo ./run.sh shm_mgr ./cfg/ae_simple_dpu.cfg

# Launch the CPU gateway
sudo ./run.sh cpu_gateway ./cfg/ae_simple_dpu.cfg

# Start the first function in the chain
sudo ./run.sh nf 1
```

---

### Worker Node 2

```bash
cd nadino-network-engine

# Start the shared memory manager
sudo ./run.sh shm_mgr ./cfg/ae_simple_dpu.cfg

# Launch the CPU gateway
sudo ./run.sh cpu_gateway ./cfg/ae_simple_dpu.cfg

# Start the second function in the chain
sudo ./run.sh nf 2
```

---

Test with curl (gateway listens on port 8080):

```bash
curl http://10.10.1.1:8080/
```

**Notes:**

* Replace `./cfg/ae_simple_dpu.cfg` with your own configuration file if needed.
* The `nf` argument specifies the function ID (`1` for the first function, `2` for the second).
* Ensure both worker nodes can reach each other over the configured RDMA network.
