# Setting up DevContainers and enable USB device access from the container

When using Windows Subsystem for Linux (WSL) the following applies:

- WSLv2 is required
- Mapping of USB into WSL requires [usbipd-win](https://github.com/dorssel/usbipd-win) - see <https://learn.microsoft.com/en-us/windows/wsl/connect-usb>
- A Docker install. Here we will be using Rancher Desktop

## Preparations

## Windows 10

This sections describes what you need to install on the host operating system, in our case Windows 10.

Perform or validate the following tasks in order

### Enable WSL2

In windows features select:

- Windows Subsystem for Linux
- Virtual Machine Platform
- Windows Hypervisor Platform

  __Reboot__ !

### Configure WSLv2

In a command Prompt as Administrator:

- `wsl --update`
- `wsl --set-default-version 2`
- `wsl --install -d Ubuntu-24.04`

After setting WSL user name and password:

- `sudo apt update; sudo apt upgrade -y`

### Install Rancher Desktop

Download and install Rancher Desktop from <https://github.com/rancher-sandbox/rancher-desktop/releases>
Launch Rancher Desktop when installation is complete. During this Proof of Concept (POC) we made these selections:

- Kubernetes not enabled
- Using dockerd (moby) as container engine

At this point you should be able to open a new command prompt and run `docker run hello-world`

### Add extensions to VS Code and test devcontainer

Add as a minimum these 2 extensions - or verify they are already installed

- WSL - identifier: `ms-vscode-remote.remote-wsl`
- Dev Containers - identifier: `ms-vscode-remote.remote-containers`

#### Test Dev Container in WSL

In VS Code:  clone this repository: <https://github.com/microsoft/vscode-remote-try-node.git> and open it. When prompted Select "open in Container"

If that __fails__, try the following:
VS Code menu File -> Close Remote and then Close folder.
Then:
VS Code menu File -> Preferences -> Settings

Search your settings for "mountWaylandSocket" and uncheck the box (this will add the directive to your settings.json file), Source <https://github.com/rancher-sandbox/rancher-desktop/issues/4735>

Try again. During testing this was the only issue I had during setup.

If all is well - close the remote and close the folder.

### Windows install usbipd-win

Using Usbipd-win is Microsoft recommended way to attach an Usb device in WSL. <https://learn.microsoft.com/en-us/windows/wsl/connect-usb>

Usbpid-win can expose an USB device as a TCP socket which can be shared to WSL - and the is a requirement for mounting the device in the container - so go ahead, and install, reboot if the installer asks for it.

First download and install `usbipd-win_4.1.0.msi` (or later) from <https://github.com/dorssel/usbipd-win/releases/latest>

The attach your device to your WSL 2 distribution using a command prompt as `Administrator`, in our case we're attaching to the Ubuntu-24.04 instance.

#### Test Usbipd with WSL

`usbipd list` - shows USB devices known by Windows:

```shell
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
2-6    0c45:6732  Integrated Webcam, Integrated IR Webcam                       Not shared
2-9    27c6:63ac  Goodix MOC Fingerprint                                        Not shared
2-10   8087:0033  Intel(R) Wireless Bluetooth(R)                                Not shared
3-3    0b0e:245e  Jabra Link 370, USB Input Device                              Not shared
3-4    046d:c548  Logitech USB Input Device, USB Input Device                   Not shared
3-5    0bda:1100  USB Input Device                                              Not shared
4-2    0bda:8153  Realtek USB GbE Family Controller                             Not shared
```

Now we can `bind` (share) devices with WSL - for the POC we will bind the Webcam so:

- `usbipd bind --busid 2-6`
- `usbipd attach --wsl --busid 2-6`

##### Usbipd troubles

You may be stopped by:

`usbipd: warning: A third-party firewall may be blocking the connection; ensure TCP port 3240 is allowed`

when you try to `usbipd attach ...`

It is caused by firewall on the network so if you cannot convince IT to allow TCP port 3240 then you can temporarily disable network interfaces:

```powershell
Get-NetAdapter | ForEach-Object { $_ | Disable-NetAdapter -confirm:$false }
```

then rerun the  `usbipd attach ...`

and finally enable networks again

```powershell
Get-NetAdapter | ForEach-Object { $_ | Enable-NetAdapter  }
```

## WSL installation commands

These commands needs to be run inside your current WSL instance, in our case we're doing it with our `Ubuntu-24.04` instance. This is also the instance that is used when you create a dev container inside Visual Studio Code.

- `sudo apt install usbutils`
- `sudo apt install linux-tools-virtual hwdata`

## Test commands

Taking pictures with `fswebcam`. Change device where appropriate

```bash
apt install fswebcam -y
sudo fswebcam -r 640x480 --jpeg 100 -D 1 hej.jpg -d /dev/bus/usb/002/002
```

### Mounting in USB to DevContainer

In your .devcontainer.json file, you need to add the following mountpoint:

```json
{
...

	"mounts": ["type=bind,source=/dev/bus/usb,target=/dev/bus/usb"]

...
}

```

### GUI for USB-PID

If you would like a gui, and also enable auto connect, then install the following on your Windows OS:

https://gitlab.com/alelec/wsl-usb-gui
