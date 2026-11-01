
# Raspberry Pi OS and ROS 2 Installation_v1

**Version 1, November 2026**

This document explains how to install Ubuntu and ROS 2 on a new Raspberry Pi so it can serve as the robot's onboard computer. When finished, the Pi runs Ubuntu Server 24.04 and ROS 2 Jazzy with SLAM Toolbox, Nav2, the project workspace, and the RPLIDAR C1 driver. Networking between the Pi and the laptop is covered in *Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1*, and viewing data on the laptop is covered in *RViz Visualization Setup_v1*.

## What You Need

Start with a Raspberry Pi 4 or 5 and its official power supply. A weak supply can cause random crashes and LiDAR dropouts. You also need a microSD card of 32 GB or larger, a computer with Raspberry Pi Imager installed, and a monitor, keyboard, and micro-HDMI cable for the first boot. For the LiDAR, you need the RPLIDAR C1, its green USB adapter board, and a USB-A to USB-C data cable.

## Step 1: Flash Ubuntu Server 24.04

Use Ubuntu Server 24.04 LTS (64-bit). ROS 2 Jazzy targets Ubuntu 24.04, and the Pi runs headless, so it doesn't need a desktop.

Open Raspberry Pi Imager. For the device, choose the Pi model. For the OS, choose Other general-purpose OS, then Ubuntu, then Ubuntu Server 24.04 LTS (64-bit). For storage, choose the microSD card.

Click Next, then Edit settings. Set the hostname to `capstone-pi` so the new Pi is easy to tell apart from the original. Choose a username and password; the original Pi uses the username `wyatt`, so substitute your own username in every command that uses it. Set the time zone to `America/Phoenix`. On the Services tab, enable SSH with password authentication. Write the card, then insert it into the Pi.

## Step 2: First Boot and Internet Connection

Do this step with a monitor and keyboard attached. The first boot takes a few minutes. Log in with the username and password set in Imager.

The Pi needs internet access before anything else can be installed. Connect it to campus Wi-Fi by following Section 1 of *Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1*, then return here. Once connected, update the system and install the basic tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl nano
```

The `build-essential` package provides the C++ compiler. Without it, the LiDAR driver build fails with the error "No CMAKE_CXX_COMPILER could be found."

## Step 3: Install ROS 2 Jazzy

The Pi uses `ros-base`, which leaves out the graphical tools, because RViz2 runs on the laptop instead. First, enable the Ubuntu universe repository:

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository universe
```

Add the ROS 2 package source, entering each line separately:

```bash
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo $VERSION_CODENAME)_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
sudo apt update && sudo apt upgrade -y
```

Install ROS 2, the build tools, and the robot packages:

```bash
sudo apt install -y ros-jazzy-ros-base ros-dev-tools
sudo apt install -y ros-jazzy-slam-toolbox ros-jazzy-navigation2 ros-jazzy-nav2-bringup ros-jazzy-robot-state-publisher ros-jazzy-joint-state-publisher ros-jazzy-xacro ros-jazzy-teleop-twist-keyboard
```

Make ROS 2 load in every new terminal, then confirm the installation:

```bash
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
source ~/.bashrc
ros2 --help
```

If the installation worked, the last command prints a list of available commands. If a terminal later reports `ros2: command not found`, ROS 2 isn't loaded in it; run `source /opt/ros/jazzy/setup.bash`.

## Step 4: Workspace and Robot Package

Create the workspace:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
```

Copy the robot package into `~/ros2_ws/src`, either from the original Pi's SD card or from the team's Git repository. Keeping the package in a shared Git repository turns future rebuilds into a single `git clone`.

## Step 5: RPLIDAR C1 Driver

From `~/ros2_ws/src`, download Slamtec's driver:

```bash
git clone https://github.com/Slamtec/sllidar_ros2.git
```

Install dependencies and build the workspace. The `rosdep init` command only needs to run once, and an "already exists" error from it can be ignored. The build takes a few minutes on the Pi.

```bash
cd ~/ros2_ws
sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -y
colcon build --symlink-install
```

If the build fails with "No CMAKE_CXX_COMPILER could be found," install `build-essential`, delete `~/ros2_ws/build/sllidar_ros2`, and build again.

Make the workspace load in every new terminal, and give the user access to USB serial devices so the driver can reach the LiDAR:

```bash
echo 'source ~/ros2_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
sudo usermod -aG dialout $USER
```

Log out and back in for the USB permission to take effect.

## Step 6: Test the LiDAR on the Pi

Connect the C1 to the Pi through its green USB adapter board, then check that the Pi detects it:

```bash
ls /dev/ttyUSB*
```

This should show `/dev/ttyUSB0`. If nothing appears even though the LiDAR spins, the cable is probably charge-only or the adapter board isn't connected; use a data cable through the adapter board. Start the driver:

```bash
ros2 launch sllidar_ros2 sllidar_c1_launch.py
```

The LiDAR should spin and the terminal should print the device information. If the driver reports "permission denied," confirm that `groups` lists `dialout`; if it doesn't, repeat the `usermod` command and log back in. Viewing the scan is covered in *RViz Visualization Setup_v1*.

## Shortcut: Moving an Existing SD Card

Everything in this document is stored on the SD card. If a working card already exists, moving it into a new Pi is much faster than reinstalling. Shut the old Pi down safely, move the card, and power on the new Pi. Ubuntu 24.04's Raspberry Pi image runs on both the Pi 4 and the Pi 5, and the workspace and settings come along with the card. Before running two Pis from copies of the same card, see the note on cloned cards in *Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1*.

## Safe Shutdown

Cutting the power while Ubuntu is running can corrupt the SD card. Press Ctrl + C in every window that's running something, then run `sudo shutdown -h now`. Wait about 20 seconds, until the green activity light stops flickering, before unplugging the power. The red power light may stay on, which is normal. To restart instead, use `sudo reboot`.

## References

The ROS 2 installation steps follow the [official ROS 2 Jazzy installation guide for Ubuntu](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html), which is worth checking for updates before each install. The LiDAR driver comes from Slamtec's [sllidar_ros2 repository](https://github.com/Slamtec/sllidar_ros2).

---
---

---
---


# RViz Visualization Setup_v1

**Version 1, November 2026**

This document explains how to set up RViz2 on the Windows laptop and use it to view LiDAR data published by the Raspberry Pi. It assumes the Pi is set up according to *Raspberry Pi OS and ROS 2 Installation_v1* and that the two machines can communicate according to *Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1*.

## How the Pieces Fit Together

The Pi does the robot's work: it runs the LiDAR driver and publishes scans on the `/scan` topic. The laptop runs RViz2 inside WSL2 Ubuntu and displays whatever the Pi publishes. Throughout this document, Terminal 1 is PowerShell with an SSH session into the Pi, and Terminal 2 is WSL2 Ubuntu on the laptop.

## Step 1: Install ROS 2 Jazzy Desktop on the Laptop

The laptop needs Windows 11 with WSL2 running Ubuntu 24.04. Inside WSL2 Ubuntu, add the ROS 2 package source exactly as in Step 3 of *Raspberry Pi OS and ROS 2 Installation_v1*, but install the desktop version, which includes RViz2:

```bash
sudo apt install -y ros-jazzy-desktop ros-dev-tools
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

Confirm that RViz2 opens by running `rviz2`. A window with a dark 3D grid should appear.

## Step 2: A Note on the ROS 2 Daemon

On this laptop, the ROS 2 command-line daemon fails to connect, so `ros2 topic` and `ros2 node` commands can hang or end with a long error ending in "Connection timed out." Always add `--no-daemon` to those commands on the laptop. RViz2 and launch files don't use the daemon and aren't affected.

## Step 3: Test With the Fake LiDAR Demo

The fake LiDAR demo, `fake_lidar_demo.py`, simulates a robot driving around a virtual room and publishes full-size 360-point scans at 10 Hz. It confirms that the whole display pipeline works before real hardware is involved. Copy it to the Pi from Windows PowerShell, replacing `PI_TAILSCALE_IP` with the Pi's Tailscale address:

```powershell
scp $env:USERPROFILE\Downloads\fake_lidar_demo.py wyatt@PI_TAILSCALE_IP:~/
```

In Terminal 1, start the demo:

```bash
python3 ~/fake_lidar_demo.py
```

In Terminal 2, confirm that scans are arriving. The rate should be about 10 Hz; press Ctrl + C to stop it.

```bash
ros2 topic hz /scan --no-daemon
```

Open `rviz2`. Under Global Options, click the value next to Fixed Frame, delete the existing text completely, type `odom`, and press Enter. Click Add, open the By topic tab, choose `/scan` and then LaserScan, and click OK. Add `/robot_marker` as a Marker display the same way, and add a TF display from the By display type tab.

Expand the LaserScan display by clicking the small triangle to its left. Double-click a value to edit it. Set Size (m) to `0.05`, set Decay Time to `60` so the room fills in as the robot drives, and set Color Transformer to `Intensity` so walls, furniture, pillars, and the moving person show in different colors. A blue robot with an orange heading arrow should drive around a room drawn in dots. Setting Decay Time back to `0` shows only what the robot sees at that moment.

Save this layout with File, then Save Config As, naming it `demo.rviz` in the home folder. Next time, open it directly with `rviz2 -d ~/demo.rviz`. Stop the demo with Ctrl + C in Terminal 1.

## Step 4: View the Real LiDAR

In Terminal 1, start the LiDAR driver as described in Step 6 of *Raspberry Pi OS and ROS 2 Installation_v1*:

```bash
ros2 launch sllidar_ros2 sllidar_c1_launch.py
```

In Terminal 2, confirm that `/scan` appears in `ros2 topic list --no-daemon`, then open `rviz2`. Set the Fixed Frame to `laser`, which is the frame the LiDAR driver uses. Add a LaserScan display on `/scan` and set its Size (m) to `0.05`. The room's walls should appear as a ring of dots around the center, and they should move when a hand passes near the LiDAR. Save this layout as `lidar.rviz` so it can be reopened with `rviz2 -d ~/lidar.rviz`.

## Troubleshooting

**RViz2 reports that the Fixed Frame doesn't exist.** The Fixed Frame doesn't match the data. Use `laser` for the LiDAR alone and `odom` for the fake demo, and make sure the old text was fully deleted, so the box doesn't read something like `maplaser`.

**`/scan` is listed, but no dots appear.** Expand LaserScan, then Topic, and set Reliability Policy to Best Effort.

**RViz2 prints "extrapolation into the past" or "queue is full" for a few seconds at startup.** Scans arrived before the robot's position data. This is normal and can be ignored. If the errors never stop, run `date` on both machines at the same moment; a difference of more than a second means the clocks need to be synchronized.

**`/scan` doesn't appear on the laptop at all.** This is a network issue. Run the basic message test in Section 5 of *Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1*.

**The demo test passes, but real scans don't arrive.** Check that the driver is still running in Terminal 1 without errors, and that `ls /dev/ttyUSB*` on the Pi shows the LiDAR.

---
---


---
---


# Wi-Fi Static IP / Peer-to-Peer Network Configuration_v1

**Version 1, November 2026**

This document explains how the Raspberry Pi and the Windows laptop are networked so they can communicate through ROS 2 on GCU's campus Wi-Fi without a monitor attached to the Pi. It covers the Pi's Wi-Fi configuration, the laptop's WSL2 networking, Tailscale for stable addresses, ROS 2 static peers, and the Windows firewall.

## Why This Configuration Is Needed

The campus network causes three problems. The Pi receives a different school-assigned IP address depending on the campus, so its address can't be known in advance without a monitor. The Pi and laptop can also end up on different campus subnets, and ROS 2's automatic discovery only searches within its own subnet, so the two machines never find each other. Finally, the WSL2 firewall on the laptop blocks incoming traffic by default.

The solution gives each device a stable address through Tailscale, tells ROS 2 exactly where the other device is with `ROS_STATIC_PEERS`, and allows the Pi's traffic through the laptop's firewall. The current addresses are 100.113.83.104 for the laptop and 100.95.217.42 for the original Pi. A new Pi receives its own address in Section 3.

## Section 1: Campus Wi-Fi on the Pi

The campus network is LOPES, which is open and has no password. With a monitor and keyboard attached, open the Pi's network configuration file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace its contents with the following, using spaces only and no tabs:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      optional: true
      dhcp4: true
  wifis:
    wlan0:
      optional: true
      dhcp4: true
      regulatory-domain: "US"
      access-points:
        "LOPES":
          auth:
            key-management: "none"
```

Save with Ctrl + O, Enter, and Ctrl + X. Then check and apply the configuration:

```bash
sudo netplan generate
sudo netplan apply
```

If `generate` prints nothing, the file has no errors. The `dhcp4: true` setting lets the school network assign the Pi an address automatically, and the Pi reconnects to LOPES on every boot. Check the current campus address, which is the part before the `/`:

```bash
ip -4 -br addr show wlan0
```

Test SSH once from the laptop's PowerShell with `ssh wyatt@THAT_ADDRESS`. After Section 3, the Tailscale address replaces the campus address permanently.

## Section 2: WSL2 Networking on the Laptop

WSL2 must use mirrored networking mode so that WSL2 Ubuntu shares the laptop's network connection instead of sitting behind a private internal network. In PowerShell, create the WSL configuration file:

```powershell
Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value "[wsl2]`nnetworkingMode=mirrored"
```

If a `.wslconfig` file already exists, add the `networkingMode=mirrored` line under its `[wsl2]` heading instead of overwriting it. Restart WSL with `wsl --shutdown`, reopen Ubuntu, and confirm the setting with `wslinfo --networking-mode`, which should print `mirrored`. Mirrored mode requires Windows 11 22H2 or newer.

## Section 3: Tailscale

Tailscale creates a private network across devices logged into the same account and gives each device a `100.x` address that stays the same on any Wi-Fi network. These addresses only work for devices on the team's Tailscale account, so they're safe to record in documentation.

Install Tailscale on the laptop from the Tailscale website and log in with the team's account. On the Pi, install and start it, then open the printed link and log in with the same account:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Get the Pi's Tailscale address and record it, then confirm that Tailscale starts automatically on boot. The second command should print `enabled`.

```bash
tailscale ip -4
sudo systemctl is-enabled tailscaled
```

From now on, the Pi is reachable from the laptop at either campus with no monitor:

```powershell
ssh wyatt@PI_TAILSCALE_IP
```

Tailscale can't connect the Pi to Wi-Fi if the campus network refuses the connection. The Pi still has to join LOPES on its own first.

## Section 4: ROS 2 Static Peers and the Firewall

`ROS_STATIC_PEERS` tells ROS 2 the exact address of the other machine, so discovery works even when the two devices are on different subnets. On the Pi, point ROS 2 at the laptop:

```bash
echo 'export ROS_STATIC_PEERS=100.113.83.104' >> ~/.bashrc
echo 'export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET' >> ~/.bashrc
source ~/.bashrc
```

On the laptop, add the same settings to WSL2 Ubuntu's `~/.bashrc`, pointing at the Pi. To reach more than one Pi, separate the addresses with a semicolon:

```bash
export ROS_STATIC_PEERS="100.95.217.42;PI_TAILSCALE_IP"
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
```

Then allow the Pi through the WSL Hyper-V firewall. Open PowerShell as administrator. When creating the rule for the first time, use the first command; when adding a Pi to an existing rule, use the second, listing every Pi address that should be allowed:

```powershell
New-NetFirewallHyperVRule -Name ROS2FromPi -DisplayName "ROS 2 Raspberry Pi" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol UDP -RemoteAddresses 100.95.217.42 -Action Allow
Set-NetFirewallHyperVRule -Name ROS2FromPi -RemoteAddresses 100.95.217.42,PI_TAILSCALE_IP
```

This allows only UDP traffic from the Pi, rather than opening the firewall to everything. Close and reopen all terminals so the new settings load, then confirm them with `echo $ROS_STATIC_PEERS` on each machine.

## Section 5: Testing Communication

In Terminal 1 (SSH into the Pi), publish a test message:

```bash
ros2 topic pub -r 1 /pi_test std_msgs/msg/String "{data: 'Hello from Raspberry Pi'}"
```

In Terminal 2 (WSL2 Ubuntu), listen for it:

```bash
ros2 topic echo /pi_test std_msgs/msg/String --no-daemon
```

The test passes when `data: Hello from Raspberry Pi` appears once every second. Stop both with Ctrl + C. Full-size scan messages are tested in *RViz Visualization Setup_v1*.

## Moving Between Campuses

Power on the Pi without a monitor and give it a minute or two to join LOPES and Tailscale. Connect the laptop to Wi-Fi and make sure Tailscale shows as connected. Then SSH to the Pi's Tailscale address. Nothing needs to be updated when the campus address changes.

## Cloned SD Cards

Never run two Pis from copies of the same SD card at the same time, because they would share one Tailscale identity and conflict. When cloning a card for a second Pi, run `sudo tailscale logout` and then `sudo tailscale up` on the copy so it registers as a new device, and remove any unused devices from the Tailscale admin page.

## Troubleshooting

**SSH to the Tailscale address times out.** Wait another minute for the Pi to finish booting, then run `tailscale status` on the laptop to see whether the Pi is online. If it isn't, the Pi likely failed to join LOPES; attach a monitor and check `ip -4 -br addr show wlan0`.

**The laptop's `ros2` commands hang or end in "Connection timed out."** The laptop's ROS 2 daemon can't connect. Add `--no-daemon` to `ros2 topic` and `ros2 node` commands. The same problem can make it look like messages aren't arriving, with `ros2 topic info /pi_test -v` on the Pi showing a subscription count of 0. In that case, the laptop's echo command crashed before it subscribed, and `--no-daemon` fixes it.

**The topic appears in `ros2 topic list --no-daemon`, but no messages arrive.** Check that the Pi's address is included in the `ROS2FromPi` firewall rule from Section 4.

**No topics from the Pi appear at all.** Confirm that `echo $ROS_STATIC_PEERS` shows the other machine's Tailscale address on both sides, and that both terminals were reopened after the change.

**An old `ROS_DISCOVERY_SERVER` value keeps appearing.** It was usually set by hand in an earlier terminal session. Open a new terminal and check the environment with `printenv | grep ROS_`. Nothing in this setup uses a discovery server.

---
---



---
---
