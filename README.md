# Linux VPC Builder (`vpcctl`)

`vpcctl` is a command-line tool written in Python to build and manage Virtual Private Clouds (VPCs) on a single Linux host. It uses native Linux networking tools (network namespaces, bridges, veth pairs, and `iptables`) to simulate a cloud networking environment from first principles.

## Prerequisites

* A Linux machine
* `sudo` (root) privileges
* Python 3
* `iproute2` and `iptables` (standard on most Linux distros)
* The `vpcctl` script, made executable (`chmod +x vpcctl`)

## How to Use

This tool is controlled by **environment variables**. You must `export` the required variables before running a command.

**Important:** You must use `sudo -E` to preserve these variables when running the script.

```bash
# Example:
export VPC_NAME="my-vpc"
export CIDR_BLOCK="10.0.0.0/16"
sudo -E ./vpcctl create-vpc
```

---

## 🚀 Full Test Plan & Walkthrough

**IMPORTANT:** Before you start, find your host's main internet interface (e.g., `eth0`, `wlan0`) by running `ip route | grep default`. Use this value for the `INTERNET_INTERFACE` variable.

### 0. Fresh Start
Always begin with a clean slate.

```bash
sudo ./vpcctl cleanup-all
```

---

### 1. Setup VPC-A (Public + Private)

First, we'll create our main VPC and two subnets: one public (with internet) and one private (isolated).

```bash
# 1. Create the first VPC
export VPC_NAME="vpc-a"
export CIDR_BLOCK="10.0.0.0/16"
sudo -E ./vpcctl create-vpc

# 2. Add a public subnet
# (Change 'eth0' to your host's internet interface)
export VPC_NAME="vpc-a"
export SUBNET_NAME="pub-a"
export SUBNET_CIDR="10.0.1.0/24"
export SUBNET_TYPE="public"
export INTERNET_INTERFACE="eth0"
sudo -E ./vpcctl create-subnet

# 3. Add a private subnet
export VPC_NAME="vpc-a"
export SUBNET_NAME="priv-a"
export SUBNET_CIDR="10.0.2.0/24"
export SUBNET_TYPE="private"
sudo -E ./vpcctl create-subnet
```

---

### 2. Test VPC-A Connectivity

Let's test our new subnets.

```bash
# Test 1: Ping the private subnet's gateway (should work)
# This proves the subnet is connected to the bridge.
export SUBNET_NAME="priv-a"
sudo -E ./vpcctl exec ping -c 2 10.0.2.1

# Test 2: Test public subnet internet access (should work)
# This proves our NAT rule is working.
export SUBNET_NAME="pub-a"
sudo -E ./vpcctl exec ping -c 2 8.8.8.8

# Test 3: Test private subnet internet access (should FAIL)
# This proves our private subnet is correctly isolated.
export SUBNET_NAME="priv-a"
sudo -E ./vpcctl exec ping -c 2 8.8.8.8
```

---

### 3. Setup VPC-B & Test Isolation

Now, let's create a *second* VPC to prove it's isolated from `vpc-a`.

```bash
# 1. Create the second VPC
export VPC_NAME="vpc-b"
export CIDR_BLOCK="10.1.0.0/16"
sudo -E ./vpcctl create-vpc

# 2. Add a public subnet to VPC-B
# (Change 'eth0' to your host's internet interface)
export VPC_NAME="vpc-b"
export SUBNET_NAME="pub-b"
export SUBNET_CIDR="10.1.1.0/24"
export SUBNET_TYPE="public"
export INTERNET_INTERFACE="eth0"
sudo -E ./vpcctl create-subnet

# 3. Test Isolation (should FAIL)
# Try to ping pub-a's server (10.0.1.10) from pub-b.
# This fails because our main firewall rule (FORWARD DROP) blocks it.
export SUBNET_NAME="pub-b"
sudo -E ./vpcctl exec ping -c 2 10.0.1.10
```

---

### 4. Deploy Apps & Test Intra-VPC Routing

Let's test if subnets *inside* the same VPC can talk to each other.

```bash
# 1. Deploy a web server in pub-a (in a new terminal)
# (This command will hang as the server runs)
export SUBNET_NAME="pub-a"
sudo -E ./vpcctl exec python3 -m http.server 80

# 2. Deploy a web server in priv-a (in a third terminal)
# (This command will also hang)
export SUBNET_NAME="priv-a"
sudo -E ./vpcctl exec python3 -m http.server 80

# 3. Test Intra-VPC routing (should work)
# From your original terminal, curl pub-a from priv-a.
export SUBNET_NAME="priv-a"
sudo -E ./vpcctl exec curl [http://10.0.1.10](http://10.0.1.10)

# 4. Test Inter-VPC isolation with curl (should FAIL)
# This proves again that vpc-b cannot reach vpc-a.
export SUBNET_NAME="pub-b"
sudo -E ./vpcctl exec curl -m 3 [http://10.0.1.10](http://10.0.1.10)
```

---

### 5. Test VPC Peering

Now we'll explicitly connect the two VPCs.

```bash
# 1. Enable peering between vpc-a and vpc-b
export VPC1_NAME="vpc-a"
export VPC2_NAME="vpc-b"
sudo -E ./vpcctl peer-vpcs

# 2. Re-run the isolation test (should work)
# The curl should now succeed because peering is enabled.
export SUBNET_NAME="pub-b"
sudo -E ./vpcctl exec curl [http://10.0.1.10](http://10.0.1.10)
```

---

### 6. Test Firewall (Security Groups)

Let's block port 80 traffic from `vpc-b`, even though they are peered.

```bash
# 1. Create the firewall rules file
cat <<EOF > rules.json
{
  "ingress": [
    {"port": 80, "protocol": "tcp", "action": "deny", "source": "10.1.0.0/16"},
    {"port": 22, "protocol": "tcp", "action": "allow", "source": "0.0.0.0/0"}
  ]
}
EOF

# 2. Apply the firewall rules to pub-a
export SUBNET_NAME="pub-a"
export RULES_FILE="rules.json"
sudo -E ./vpcctl apply-firewall

# 3. Re-run the peering test (should FAIL)
# The curl fails because the firewall inside ns-pub-a blocks it.
export SUBNET_NAME="pub-b"
sudo -E ./vpcctl exec curl -m 3 [http://10.0.1.10](http://10.0.1.10)
```

---

### 7. Cleanup

Finally, tear down all resources.

```bash
# 1. List all resources created
sudo ./vpcctl list

# 2. Run the master cleanup command
sudo ./vpcctl cleanup-all

# 3. Verify everything is gone
sudo ./vpcctl list
```