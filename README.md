# nimbuspwn

This is a PoC for Nimbuspwn, as originally described in https://www.microsoft.com/security/blog/2022/04/26/microsoft-finds-new-elevation-of-privilege-linux-vulnerability-nimbuspwn/

It runs reliably on Ubuntu Desktop installs, but does not run by default on Ubuntu Server installs. It is possible to configure a server install to be vulnerable, although this is not expected to be a common configuration.

## Making an AWS Instance Vulnerable

This vulnerability is generally not present in pure `systemd-networkd` based systems, since in that case, the legitimate `systemd-networkd` process will be holding the required D-Bus name. However, it is present where attempts have been made to disable `systemd-networkd`, for example using scripts such as https://gist.github.com/polrus/772618cfead9c1b63b246584024d7765, which enables `NetworkManager` for Ubuntu Server installs.

### Commands

````
sudo apt update
sudo apt -y install network-manager
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
Add content: "network: {config: disabled}" and save
sudo nano /etc/netplan/50-cloud-init.yaml
Add "renderer: NetworkManager" to "network:" block
sudo netplan generate
sudo netplan apply
sudo systemctl enable NetworkManager.service
sudo systemctl restart NetworkManager.service
sudo systemctl disable systemd-networkd
````

# Detection
To aid detection of this specific poc we have released a sigma rule. It should be noted that this sigma rule is specific to our poc code and may not detect all possible exploit attempts.

# License

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.





#######################################################################################3
#######################################################################################3
#######################################################################################3
#######################################################################################3



# Nimbuspwn detector

### Overview

This tool performs several tests to determine whether the system is possibly vulnerable to [Nimbuspwn](https://www.microsoft.com/security/blog/2022/04/26/microsoft-finds-new-elevation-of-privilege-linux-vulnerability-nimbuspwn/) (CVE-2022-29799 & CVE-2022-29800), a vulnerability in the `networkd-dispatcher` daemon discovered by the Microsoft 365 Defender Research Team.

A system is deemed possibly vulnerable to exploitation if the following conditions are met:
1. The vulnerable service `networkd-dispatcher` service is running.
2. The `systemd-networkd` service is either not running or not set to run at next boot. Since this service owns the `org.freedesktop.network1` bus on startup, an attacker will not be able to send messages on the bus if this service is running.
3. The `systemd-network` user is in use. Specifically whether a process owned by this user is running, or that there exist [setuid](https://www.liquidweb.com/kb/how-do-i-set-up-setuid-setgid-and-sticky-bits-on-linux/)-executables owned by this user. An attacker must run code as the `systemd-network` user in order to own the `org.freedesktop.network1` bus name and exploit the vulnerability. The attacker may be able to subvert these processes and/or setuid-executables to run arbitrary code. **Note that the existence of such processes or binaries does not guarantee they can be subverted for arbitrary code execution by an attacker**.

### Usage
```
./nimbuspwn-detector.sh [--full-suid]
```

The tool will check for the preconditions mentioned in the last section.
When the `--full-suid` flag is not given, relevant setuid-executables will be searched recursively under the  `/sbin` and `/usr/sbin` directories only.
When the `--full-suid` flag is given, the search is performed recursively on the entire root volume (`/`).
