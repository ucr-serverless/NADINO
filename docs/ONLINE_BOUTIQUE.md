## Online Boutique Workload

This experiment demonstrates deploying all data-plane components—**Ingress** and the **Network Engine**—to run the **Online Boutique** microservices application.

---

### DPU-Based Network Engine Setup

> ⚙️ Use configuration file: `./cfg/online-boutique-palladium-dpu.cfg`

Start components in the following order:

1. Memory manager on worker node 1
2. Sockmap manager on worker node 1
3. Memory manager on worker node 2
4. Sockmap manager on worker node 2
5. Network Engine on **DPU1** (attached to worker node 1)
6. Network Engine on **DPU2** (attached to worker node 2)
7. Ingress on the ingress node
8. Functions on worker node 1
9. Functions on worker node 2

**Worker Node 1:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-dpu.cfg
sudo ./run.sh sockmap_manager
sudo ./run.sh frontendservice 1
sudo ./run.sh recommendationservice 5
sudo ./run.sh checkoutservice 7
```

**DPU1 (network engine):**

```bash
sudo ./run.sh gateway ./cfg/online-boutique-palladium-dpu.cfg
```

**Worker Node 2:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-dpu.cfg
sudo ./run.sh sockmap_manager
sudo ./run.sh currencyservice 2
sudo ./run.sh productcatalogservice 3
sudo ./run.sh cartservice 4
sudo ./run.sh shippingservice 6
sudo ./run.sh paymentservice 8
sudo ./run.sh emailservice 9
sudo ./run.sh adservice 10
```

**DPU2 (network engine):**

```bash
sudo ./run.sh gateway ./cfg/online-boutique-palladium-dpu.cfg
```

**Load Generator:**

* Use `wrk` for load generation:

  ```bash
  # Home Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/

  # View Cart
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/cart

  # Product Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/product?1YMWWN1N4O
  ```

---

### CPU-Based Network Engine Setup

> ⚙️ Use configuration file: `./cfg/online-boutique-palladium-dpu.cfg`

Start components in the following order:

1. Memory manager on worker node 1
2. Memory manager on worker node 2
3. Network Engine on worker node 1
4. Network Engine on worker node 2
5. Ingress on the ingress node
6. Functions on worker node 1
7. Functions on worker node 2

**Worker Node 1:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-dpu.cfg
sudo ./run.sh frontendservice 1
sudo ./run.sh recommendationservice 5
sudo ./run.sh checkoutservice 7
```

**Worker Node 2:**

```bash
sudo ./run.sh shm_mgr ./cfg/online-boutique-palladium-dpu.cfg
sudo ./run.sh currencyservice 2
sudo ./run.sh productcatalogservice 3
sudo ./run.sh cartservice 4
sudo ./run.sh shippingservice 6
sudo ./run.sh paymentservice 8
sudo ./run.sh emailservice 9
sudo ./run.sh adservice 10
```

**Load Generator:**

* Use `wrk` for load generation:

  ```bash
  # Home Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/

  # View Cart
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/cart

  # Product Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<INGRESS_IP>:80/rdma/1/product?1YMWWN1N4O
  ```

---

### DNE Without Ingress

> This setup runs the **DPU-based Network Engine (DNE)** without the Ingress.

> ⚙️ Use configuration file: `./cfg/my-palladium-cpu.cfg`

Start components in the following order:

1. Start the memory manager on **worker node 1**
2. Start the memory manager on **worker node 2**
3. Start the network engine on **DPU1**
4. Start the network engine on **DPU2**
5. Launch functions on **worker node 1**
6. Launch functions on **worker node 2**

---

**Worker Node 1:**

```bash
sudo ./run.sh shm_mgr ./cfg/my-palladium-cpu.cfg
sudo ./run.sh frontendservice 1
sudo ./run.sh recommendationservice 5
sudo ./run.sh checkoutservice 7
```

---

**Worker Node 2:**

```bash
sudo ./run.sh shm_mgr ./cfg/my-palladium-cpu.cfg
sudo ./run.sh currencyservice 2
sudo ./run.sh productcatalogservice 3
sudo ./run.sh cartservice 4
sudo ./run.sh shippingservice 6
sudo ./run.sh paymentservice 8
sudo ./run.sh emailservice 9
sudo ./run.sh adservice 10
```

**Load Generator:**

* Use `wrk` for load generation:

  ```bash
  # Home Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<NETENG_IP>:8080/1/

  # View Cart
  wrk -t<num_threads> -c<num_clients> -d30s http://<NETENG_IP>:8080/1/cart

  # Product Query
  wrk -t<num_threads> -c<num_clients> -d30s http://<NETENG_IP>:8080/1/product?1YMWWN1N4O
  ```