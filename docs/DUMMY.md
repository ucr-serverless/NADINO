## Running with a Dummy Function Chain

This example demonstrates a minimal setup where two functions are chained linearly:

```
client → function-1 → function-2 → client
```

The deployment uses **two worker nodes**, each running a memory manager, gateway, and one function instance.

---

### Worker Node 1

```bash
cd nadino-network-engine

# Start the shared memory manager
sudo ./run.sh shm_mgr ./cfg/my-palladium-cpu.cfg

# Launch the gateway
sudo ./run.sh gateway ./cfg/my-palladium-cpu.cfg

# Start the first function in the chain
sudo ./run.sh nf 1
```

---

### Worker Node 2

```bash
cd nadino-network-engine

# Start the shared memory manager
sudo ./run.sh shm_mgr ./cfg/my-palladium-cpu.cfg

# Launch the gateway
sudo ./run.sh gateway ./cfg/my-palladium-cpu.cfg

# Start the second function in the chain
sudo ./run.sh nf 2
```

---

✨ **Notes:**

* Replace `./cfg/my-palladium-cpu.cfg` with your own configuration file if needed.
* The `nf` argument specifies the function ID (`1` for the first function, `2` for the second).
* Ensure both worker nodes can reach each other over the configured RDMA network.
