## Online Boutique Workload

This experiment deploys all data-plane components — **Ingress** and **Network Engine** — to run the **Online Boutique** microservices application.

For full details see the [nadino-network-engine Running section](../nadino-network-engine/README.md#running).

---

### DPU-Based Network Engine Setup (DNE)

> Use configuration file: `./cfg/ae_online-boutique-palladium-dpu.cfg`

Start components in the following order:

1. Shared memory manager on **worker1**
2. Sockmap manager on **worker1**
3. Shared memory manager on **worker2**
4. Sockmap manager on **worker2**
5. Gateway on **DPU1** (attached to worker1)
6. Gateway on **DPU2** (attached to worker2)
7. NADINO Ingress on the **ingress node** (see [nadino-ingress](../nadino-ingress/README.md))
8. Functions on **worker1**
9. Functions on **worker2**

**Worker Node 1:**

```bash
sudo ./run.sh shm_mgr ./cfg/ae_online-boutique-palladium-dpu.cfg
sudo ./run.sh sockmap_manager
sudo ./run.sh frontendservice 1
sudo ./run.sh recommendationservice 5
sudo ./run.sh checkoutservice 7
```

**DPU1:**

```bash
sudo ./run.sh gateway ./cfg/ae_online-boutique-palladium-dpu.cfg
```

**Worker Node 2:**

```bash
sudo ./run.sh shm_mgr ./cfg/ae_online-boutique-palladium-dpu.cfg
sudo ./run.sh sockmap_manager
sudo ./run.sh currencyservice 2
sudo ./run.sh productcatalogservice 3
sudo ./run.sh cartservice 4
sudo ./run.sh shippingservice 6
sudo ./run.sh paymentservice 8
sudo ./run.sh emailservice 9
sudo ./run.sh adservice 10
```

**DPU2:**

```bash
sudo ./run.sh gateway ./cfg/ae_online-boutique-palladium-dpu.cfg
```

---

### CPU-Based Network Engine Setup (CNE)

> Use configuration file: `./cfg/online-boutique-palladium-host.cfg`

Start components in the following order:

1. Shared memory manager on **worker1**
2. CPU gateway on **worker1**
3. Shared memory manager on **worker2**
4. CPU gateway on **worker2**
5. NADINO Ingress on the **ingress node**
6. Functions on **worker1**
7. Functions on **worker2**

**Worker Node 1:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-host.cfg
sudo ./run.sh cpu_gateway ./cfg/online-boutique-palladium-host.cfg
sudo ./run.sh frontendservice 1
sudo ./run.sh recommendationservice 5
sudo ./run.sh checkoutservice 7
```

**Worker Node 2:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-host.cfg
sudo ./run.sh cpu_gateway ./cfg/online-boutique-palladium-host.cfg
sudo ./run.sh currencyservice 2
sudo ./run.sh productcatalogservice 3
sudo ./run.sh cartservice 4
sudo ./run.sh shippingservice 6
sudo ./run.sh paymentservice 8
sudo ./run.sh emailservice 9
sudo ./run.sh adservice 10
```

---

### Load Generation

Use `wrk` from the load generator node (replace `<INGRESS_IP>` with the ingress node IP):

```bash
# Home page
wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/

# View cart
wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/cart

# Product query
wrk -t<num_threads> -c<num_clients> -d30s "http://<INGRESS_IP>:80/rdma/1/product?1YMWWN1N4O"
```
