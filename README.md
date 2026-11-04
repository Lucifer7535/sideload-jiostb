# Jio STB ADB Setup Guide (Windows 10 & 11)

A beginner-friendly, clutter-free walkthrough for enabling **network ADB debugging** and sideloading custom launchers and apps onto supported Jio Set-Top Boxes (STBs), using a Windows PC and your home Wi-Fi router.

This repository is a fork. All credit for the original research, tooling, and instructions belongs to the upstream project: [ilikechocopies/sideload-jiostb](https://github.com/ilikechocopies/sideload-jiostb).

The only change here is an expanded `README.md`, with more detailed steps and clearer explanations of the issues I ran into while following the original guide. I hope it helps you too.

> [!TIP]
> **Need help?**
> * **Reddit:** Reach out directly to me at [u/lucifer7535](https://www.reddit.com/user/lucifer7535)
> * **Instagram:** [@pranaypawar7535](https://instagram.com/pranaypawar7535)

> [!NOTE]
> **Supported hardware and firmware:**
> * **JMSC200Av2** and **JMSB200A** across firmware builds `22.1.09`, `24.1.39`, `25.1.9`, and older.
> * **JHS series (JHSC200 v1 / JioOS 2.0 / JHSD200)** on firmware builds `stb-jhcs200-11.3.46` and `stb-jhcsd200-11.3.46`.
>
> Both box families use the exact same `./stage1.sh` workflow. The only difference is in **Step 8**, where you launch the proxy listener that matches your model.

> [!TIP]
> **Don't panic if something goes wrong:**
> * **Box acting up?** Factory reset your Jio STB to restore the default settings, then start over.
> * **Wi-Fi not connecting properly?** Open the network settings, choose **Forget Network**, re-enter your password, and reconnect.

> [!IMPORTANT]
> **Package installer limitation (read this first):**
> Jio disabled the stock Android package installer UI on the box. Because of this, you cannot simply click an APK file on the TV to install it. **Every time you want to install an app, you must install it over ADB** from your PC:
>
> ```bash
> adb install "filename.apk"
> ```

---

## ✨ What Happens After Running the Script?

Once you execute Stage 1, the script performs several quality-of-life modifications on your Jio STB:

* 🔓 **ADB enabled:** Network ADB is activated (the authorization expires in 2037), so you can manage and sideload apps from your PC.
* 🚀 **Monet Launcher activated:** Replaces the cluttered stock interface with a clean, customizable TV launcher. The stock launcher is suspended to prevent it from reverting after a reboot.
* ⚡ **Shevery (Shizuku-compatible API) installed:** A standalone manager that lets supported apps access privileged system APIs over ADB.
* 🛑 **Automatic firmware updates blocked:** OTA auto-updates are disabled (`com.jio.*` on JMSC/JMSB and `com.iwedia.*` on JHS), so your setup is not overwritten.
* 🚫 **Startup boot ads disabled:** The commercial promo and intro ads shown when the STB powers on are suppressed.

---

## ⚡ Quick Navigation

* [Prerequisites](#-prerequisites)
* [Step-by-Step Guide](#-step-by-step-guide)
  * [1. Network Pairing](#step-1-network-pairing)
  * [2. Get the Host IP](#step-2-get-the-host-ip)
  * [3. Configure the STB DNS via the USB Trick](#step-3-configure-the-stb-dns-via-the-usb-trick)
  * [4. Run Docker](#step-4-run-docker)
  * [5. Set Up Technitium DNS](#step-5-set-up-technitium-dns)
  * [6. Deploy the TLS Containers](#step-6-deploy-the-tls-containers)
  * [7. Isolate the Network & Temporarily Disable the Firewall](#step-7-isolate-the-network--temporarily-disable-windows-firewall)
  * [8. Launch the Proxy Listener (Model Specific)](#step-8-launch-the-proxy-listener)
  * [9. Initialize Git Bash & Run a Health Check](#step-9-initialize-git-bash--health-check)
  * [10. Execute Stage 1](#step-10-execute-stage-1)
  * [11. Verify the ADB Connection](#step-11-verify-the-adb-connection)
  * [12. Teardown & Reset](#step-12-teardown--reset)
* [How to Install Apps via ADB](#-how-to-install-apps-via-adb)
* [Advanced CLI Options](#-advanced-cli-options)
* [Credits & Acknowledgments](#-credits--acknowledgments)

---

## 🛠 Prerequisites

Install these utilities on your Windows system before starting:

| Tool | Purpose | Download |
| ----- | ----- | ----- |
| **Docker Desktop** | Hosts the local GenieACS backend services | [Official download](https://docs.docker.com/desktop/setup/install/windows-install/) |
| **Python 3** | Runs the proxy and diagnostic scripts (leave **Add python.exe to PATH** checked during install) | [python.org](https://www.python.org/downloads/) |
| **Technitium DNS** | Overrides local domain resolution so the STB reaches your PC | [technitium.com](https://technitium.com/dns/) |
| **Git for Windows** | Provides the Git Bash terminal environment | [gitforwindows.org](https://gitforwindows.org/) |

> [!NOTE]
> The Android platform-tools ADB binaries are already bundled inside this folder, so you do not need to install the Android SDK separately.

> [!IMPORTANT]
> **Keep your internet connection up until Step 7.** Docker needs internet access to pull the GenieACS and TLS container images, and Technitium must be configured before you cut the router off from the WAN.

---

## 🚀 Step-by-Step Guide

### Step 1: Network Pairing

* Connect your Windows PC to your primary home Wi-Fi.
* Connect your Jio STB to the same router, either over standard Wi-Fi or Ethernet.
* **Do not use Guest Wi-Fi.** Guest networks isolate connected devices, so the STB will not be able to reach your PC.

### Step 2: Get the Host IP

Open Command Prompt (`Win + R` → `cmd`) and run:

```cmd
ipconfig
```

Find the **Wireless LAN adapter Wi-Fi** section (or **Ethernet**, if you are wired), and copy the **IPv4 Address** — for example, `192.168.29.13`. You will need this address in Steps 3, 5, and 9.

### Step 3: Configure the STB DNS via the USB Trick

> [!NOTE]
> An empty USB pendrive is required only to access the hidden stock Android settings menu. Any spare drive will do.

1. Plug the USB thumb drive into the Jio STB.
2. Turn on the STB → launch **My Music** → **Settings** → select **Eject USB**.
3. Tap **Internal Storage** → open **Apps** → select the stock **Settings** app (the gear icon).
4. Navigate to **Network** → select your Wi-Fi network → open **Advanced IP Settings**.
5. Change **IP Settings** from **DHCP** to **Static**.
6. Set an **IP Address** in the same subnet as your router. If your PC's address is `192.168.29.13`, choose an unused address in the same range — for example `192.168.29.128` (anything free on your network works).
7. Leave **Gateway**, **Subnet Mask**, and **DNS 2** unchanged.
8. Set **DNS 1** to your Windows PC's IPv4 address from Step 2.
9. Save the settings and confirm that the STB reconnects to Wi-Fi.

> [!WARNING]
> If the **Gateway** or **IP Address** fields clear themselves when you switch to Static, switch back to DHCP, note the default values, switch to Static again, and re-enter them manually before saving.

#### Alternate Method: No USB Drive Available

1. Open **Settings** on the Jio STB.
2. Select **General** → **About** → **Details**.
3. Press the `OK` button on the remote 10 times.
4. Go back twice and open **System Settings**.
5. **Android Settings** now appears as the last entry in the list. Open it.
6. Open the Wi-Fi settings → select your Wi-Fi network → **Advanced IP Settings**.
7. Change **IP Settings** from **DHCP** to **Static**.
8. Set an **IP Address** in the same subnet as your router (see Step 3, item 6), leaving **Gateway**, **Subnet Mask**, and **DNS 2** unchanged.
9. Set **DNS 1** to your Windows PC's IPv4 address from Step 2.
10. Save the settings and confirm that the STB reconnects to Wi-Fi.

### Step 4: Run Docker

Docker Desktop requires WSL 2 on Windows. If you have not enabled it yet, open **Windows PowerShell** as Administrator and run:

```powershell
wsl --install
```

Reboot when prompted, then launch **Docker Desktop** from the Start menu and wait until the engine status shows as running (the whale icon in the system tray stops animating). If Docker asks you to enable WSL 2 during setup, accept and reboot.

You can confirm the engine is up from Git Bash at any time with:

```bash
docker info
```

### Step 5: Set Up Technitium DNS

Technitium runs on your PC and answers DNS queries from the STB, pointing `acs.oss.jio.com` at your machine.

1. Open the Technitium console in your browser at [`http://localhost:5380`](http://localhost:5380). On first launch, create an admin password.
2. Go to **Zones** → click **Add Zone**:
   * **Zone Name:** `jio.com`
   * **Zone Type:** `Primary Zone`
3. Open the `jio.com` zone → click **Add Record**:
   * **Name:** `acs.oss`
   * **Type:** `A`
   * **IP Address:** *your Windows IPv4 address from Step 2*
4. Save the record and make sure the `jio.com` zone is **Enabled**.

The resulting record, `acs.oss.jio.com`, is the ACS endpoint the STB requests during boot. If it does not resolve to your PC, the STB will never contact your local GenieACS instance.

### Step 6: Deploy the TLS Containers

1. Open File Explorer and navigate to the `sideload-jiostb/tls` folder.
2. Click the address bar, type `cmd`, and press **Enter**.
3. In Command Prompt, run:

```cmd
docker compose up -d
```

4. Confirm the containers are running:

```cmd
docker compose ps
```

> [!NOTE]
> The first run pulls the container images and can take several minutes. This is the main reason Steps 4–6 must be completed before you unplug the fiber cable in Step 7.

### Step 7: Isolate the Network & Temporarily Disable Windows Firewall

1. **Unplug the incoming internet connection:**
   Unplug the **WAN / incoming fiber cable** from your router, but keep the router powered on. Both devices must be able to talk over the local network without active internet access, so the STB is forced to use your local DNS and ACS server instead of Jio's real ones.

2. **Temporarily disable Windows Defender Firewall:**
   > [!IMPORTANT]
   > Windows Firewall frequently blocks incoming connections from the STB on port **53** (DNS) and ports **8443/9443** (TLS proxy), which makes the process fail silently. Turn it off temporarily:
   >
   > 1. Press `Win + R`, type `firewall.cpl`, and press **Enter**.
   > 2. Click **Turn Windows Defender Firewall on or off** in the left menu.
   > 3. Select **Turn off Windows Defender Firewall (not recommended)** for both the *Private* and *Public* network profiles.
   > 4. Click **OK**. You will turn the firewall back on in Step 12.

### Step 8: Launch the Proxy Listener

The proxy front end terminates the TLS connection from the STB and forwards the traffic to GenieACS. Pick **Option A** or **Option B** depending on your model.

#### Option A: JMSC / JMSB Boxes (Default)

In the same Command Prompt window (inside the `tls` folder), run:

```cmd
python -m pip install pycryptodome
python pass_front.py
```

Leave this window running for the rest of the process. `pass_front.py` includes a precomputed token fallback, so it still works if `pycryptodome` is not installed.

#### Option B: JHS Boxes (JHSC200 / JioOS 2.0 / JHSD200)

JHS boxes enforce strict system CA certificate validation and reject the default certificate chain, so `pass_front.py` alone is not enough:

1. Open **Git Bash** in the main `sideload-jiostb` folder.
2. Run the automated eMudhra certificate minting script:

   ```bash
   ./jhs/run_acs.sh
   ```

   This recovers the leaked eMudhra intermediate key, mints a certificate trusted by the STB's system certificate store, and launches the proxy on ports 8443 and 9443. Leave this terminal open as well.

### Step 9: Initialize Git Bash & Health Check

1. Open the main `sideload-jiostb` folder.
2. Right-click an empty area → **Show more options** → **Open Git Bash here**.

*(Optional but highly recommended)* Prove your PC setup works before touching the TV:

```bash
python doctor.py --quick
python doctor.py --simulate
```

`--simulate` sends a synthetic Inform through the proxy and GenieACS, verifying that the entire chain is operational. A healthy run looks like this:

```text
$ python doctor.py --quick

Running Quick Chain Health Check...
[PASS] pass_front.py listening on port 8443
[PASS] GenieACS CWMP listening on port 7547
[PASS] GenieACS NBI reachable at http://127.0.0.1:7557
[PASS] DNS acs.oss.jio.com -> 192.168.29.13 (Matches host LAN IP)

$ python doctor.py --simulate

Simulating Full CWMP Chain (Synthetic Inform)...
[1/4] Sending synthetic Inform over TLS to https://127.0.0.1:8443/ ...
[PASS] Inform accepted by GenieACS! (Session cookie: session=9dbd11cb06205528)
[2/4] Sending empty POST with session cookie to close CWMP session...
[PASS] CWMP session completed cleanly (HTTP 204)
```

> [!WARNING]
> **If you see this error:** `[FAIL] DNS acs.oss.jio.com could not be resolved! Check Technitium DNS / hosts`
>
> Your PC is not using Technitium for DNS. Fix it on Windows 10 (the same setting exists on Windows 11):
>
> 1. Search Windows for **View network connections** and open it.
> 2. Right-click your Wi-Fi adapter and select **Properties**.
> 3. Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
> 4. Select **Use the following DNS server addresses**.
> 5. Enter your PC's IP address from Step 2 as the **Preferred DNS server**.
> 6. Run `python doctor.py --quick` again — it should now pass.

> [!NOTE]
> **`[FAIL] Simulated device not found in GenieACS NBI (HTTP 405)`** can be ignored; proceed to the next step.

### Step 10: Execute Stage 1

Make sure Docker, Technitium, and the proxy listener from Step 8 are all still running, then list the devices GenieACS can see:

```bash
./stage1.sh --list-devices
```

You should see two entries: the simulated test device created by `doctor.py --simulate`, and your real STB.

```text
[2/6] Finding device in GenieACS...
  1 online, 2 total:
    1) 000000-TEST%2DJioSTB-SIMULATED%2D9999  [STALE (25m ago)]  <-- Simulated device from doctor.py
    2) CCF55F-JHSD200-RFLSBBII2242025          [ONLINE]           <-- Your real STB
```

Select your STB when prompted, then run the full script:

```bash
./stage1.sh
```

> [!NOTE]
> Avoid `./stage1.sh --tasks-only` for this flow; it does not work reliably. Run the full script and let it queue the tasks itself.

> [!IMPORTANT]
> **A cold power-cycle is mandatory (unplug the wall power):**
> The Jio STB only contacts the ACS server during **boot**. Its periodic background check-in takes hours, and remote wake is impossible because `NATDetected=True` closes incoming port 7547 on the STB.
>
> 1. Unplug the STB power adapter from the wall — do not just put it in standby with the remote.
> 2. Wait 5 seconds, then plug it back in.
> 3. Let it boot up completely.

Once the STB finishes booting, it will display a prompt asking you to allow the connection. Accept it.

### Step 11: Verify the ADB Connection

> [!NOTE]
> **"Allow USB debugging?" dialog on the TV:**
> No prompt appears on your TV until the STB boots and applies the CWMP task. Once it connects, the screen displays an RSA key prompt.
> Check **Always allow from this computer** and select **OK**.
> If `adb connect` initially reports `failed to authenticate` or `unauthorized`, that is completely normal before you approve the prompt. Simply run `adb connect <STB_IP>:5555` again afterwards.

In Git Bash, connect to the static IP you assigned in Step 3 (for example `192.168.29.128`) and confirm the connection:

```bash
adb connect 192.168.29.128:5555
adb devices
```

Your device should now be listed with the status **`device`**. If the list is empty, run `adb kill-server` followed by `adb devices` to restart the ADB daemon, then try `adb connect` again.

### Step 12: Teardown & Reset

Once ADB is confirmed and your setup is complete:

1. Stop `pass_front.py` / `run_acs.sh` with `Ctrl + C` in its window, or run `./stage1.sh --stop`.
2. In Git Bash (from the `tls` folder), stop the Docker containers:

   ```bash
   docker compose down
   ```

3. Reconnect your router's incoming WAN/Internet cable.
4. **Turn Windows Defender Firewall back on:** press `Win + R` → `firewall.cpl` → **Turn Windows Defender Firewall on or off** → restore both profiles.
5. If you set a **Preferred DNS server** during Step 9, switch it back to **Obtain DNS server address automatically**.

---

## 📲 How to Install Apps via ADB

Because the built-in package installer UI is disabled on the STB, install any Android TV APK from your PC:

1. Connect your PC to the STB over ADB, if it is not already connected:

   ```bash
   adb connect <STB_IP_ADDRESS>:5555
   ```

2. Sideload your preferred application:

   ```bash
   adb install "C:\path\to\your_app.apk"
   ```

3. The app appears immediately inside **Monet Launcher**.

> [!TIP]
> If `adb install` fails with `INSTALL_FAILED_*`, add `-r` to replace an existing copy (`adb install -r "app.apk"`), and double-check that the APK is built for Android TV / your box's CPU architecture.

---

## 🧰 Advanced CLI Options

| Command | Purpose |
|---|---|
| `./stage1.sh --doctor` | Run the full diagnostic checklist for ports, NBI, and DNS |
| `./stage1.sh --simulate` | Test a synthetic CWMP session through the TLS front end and GenieACS |
| `./stage1.sh --watch` | Real-time watcher polling for the STB Inform and port 5555 |
| `./stage1.sh --tasks` | Inspect all pending TR-069 tasks queued for your STB |
| `./stage1.sh --clear-tasks` | Clear duplicate or stale tasks from the GenieACS queue |
| `./stage1.sh --stop` | Stop the background `pass_front.py` process |
| `./stage1.sh --yes-disable-tr069` | Explicitly permit disabling `com.jio.tr069client` in the script's Stage 6 |
| `./jhs/run_acs.sh` | Mint an eMudhra-forged TLS certificate and start the proxy for JHS boxes |
| `./jhs/jhs_stage1.sh --status` | Display the persisted Iwedia ADB properties and launcher state on JHS |
| `./jhs/jhs_stage1.sh --post-setup` | Sideload Monet/Shevery and disable the Iwedia OTA services on JHS |
| `./jhs/jhs_stage1.sh --clu-probe` | Queue a native root command execution probe via `tr069native` |
| `./jhs/jhs_stage1.sh --clu-read` | Read back the executed command output from the `tr069native` root probe |

---

## ⚠️ Important Disclaimers

* **Scope:** Stage 1 performs modifications such as enabling ADB, queueing downgrade flags, installing Monet Launcher and Shevery, disabling TR-069 settings, and disabling startup ads.
* **Root/Bootloader:** This guide deliberately excludes `unlock.sh` and `pull_boot.sh`. Do not run the Stage 2 utilities unless you know exactly what you are doing.
* **Liability:** Use this only on hardware you own or have explicit authorization to test.

---

## 🌟 Credits & Acknowledgments

* **Sideload Jio STB Repo**
  https://github.com/ilikechocopies/sideload-jiostb

* **Documentation, Windows 11 Streamlining & Maintenance:**
  Maintained by **Vishal Beniwal** (Instagram: [**@soloxsurfer**](https://instagram.com/soloxsurfer) / Reddit: [**u/ilikechocopies**](https://www.reddit.com/user/ilikechocopies))

* **Original Vulnerability Research & Tooling:**
  Massive respect to **Bihaan Sen**, **ak_0o7**, and **Otus9051** for the reverse-engineering and the original [Housekeeping-PoC](https://github.com/JFC-Group/Housekeeping-PoC/) repository. Please star and support their project!

* **JHS Series & eMudhra Chain Research:**
  Huge thanks to **Pavit Chhabra** ([@mesosan16](https://twitter.com/mesosan16) on Twitter) for helping add JHS series support.
