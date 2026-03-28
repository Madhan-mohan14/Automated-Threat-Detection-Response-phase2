# Step 1: VirtualBox & VM Setup

## 1.1 Install Oracle VirtualBox

Download VirtualBox from [virtualbox.org](https://www.virtualbox.org/) and install it for your host OS.

## 1.2 Create NAT Network

1. Open VirtualBox → **File** → **Tools** → **Network Manager**
2. Click the **NAT Networks** tab
3. Click **Create** and configure:
   - **Name:** `WazuhLabNet`
   - **IPv4 Prefix:** `10.0.2.0/24`
   - **Enable DHCP:** Yes
4. Click **Apply**

## 1.3 Create Ubuntu Server VM

1. Click **New** in VirtualBox
2. Configure:
   - **Name:** `Ubuntu Server`
   - **Type:** Linux → Ubuntu (64-bit)
   - **RAM:** 4096 MB (4 GB minimum, 6-8 GB recommended)
   - **CPU:** 2 cores
   - **Disk:** 30 GB (dynamically allocated)
3. Before starting, go to **Settings** → **Network**:
   - Adapter 1 → Attached to: **NAT Network**
   - Name: `WazuhLabNet`
4. Mount the Ubuntu Server 22.04 ISO and install
5. During installation:
   - Set hostname: `Maddy` (or your preferred name)
   - Create user: `maddy`
   - Enable OpenSSH Server when prompted
6. After install, note the IP address:
   ```bash
   hostname -I
   # Should show 10.0.2.4 or similar
   ```

## 1.4 Create Kali Linux VM

1. Click **New** in VirtualBox
2. Configure:
   - **Name:** `Kali Linux`
   - **Type:** Linux → Debian (64-bit)
   - **RAM:** 2048 MB (2 GB)
   - **CPU:** 2 cores
   - **Disk:** 25 GB
3. **Settings** → **Network**:
   - Adapter 1 → Attached to: **NAT Network**
   - Name: `WazuhLabNet`
4. Mount Kali Linux ISO and install with defaults
5. Verify network:
   ```bash
   ifconfig
   # eth0 should show 10.0.2.5 or similar
   ```

## 1.5 Verify Connectivity

From Kali, ping the Ubuntu server:
```bash
ping 10.0.2.4
```

From Ubuntu, ping Kali:
```bash
ping 10.0.2.5
```

Both should respond. If not, check that both VMs are on the same NAT Network.
