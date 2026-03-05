# Raspaudio - A Pi based Bluetooth audio sink with http stream output 
I own an old internet radio receiver that has neither Bluetooth nor a line in. In order to play music from my phone I wanted a device that can act as a bluetooth audio sink and forward the stream via http.
I am using a Raspberry Pi Zero 2 W with the latest Raspberry Pi OS (13.2).

## Remarks
The setup is done as the 'pi' default user, thus the 'sudo' commands. All services are however set up as system services.
The project is in a 'works for me' state and has no regards of any other scripts or services on the machine. 


## To Dos
- make the stream a service that works all the time regardless of a bluetooth audio stream present and there is no interaction needed with the system at all.
- when there is no bluetooth audio stream present, play some default sound to allow an easy check if the http stream works

## Maybe's
- turn this documentation into a setup_script.sh
- move streaming from ffmpeg to icecast/darkice
- make the device discoverable only a limited time after power on
- make a minimal web interface for managing devices (discoverable on/off, remove paired devices)
- harden the setup (readonly FS)
- make a container version

## Base Installation

Start with a Raspberry Pi Os 32 Bit (with Desktop) Version 13.2 (Trixie).

The GUI won't bee needed but the lite version might have issues with Bluetooth and/or A2DP.



## Setup the Pi for auto-login, disable GUI

Start raspi-config. Disable the GUI, enable auto-login and disable wifi energy saving.

To keep the session up any time:
```
sudo loginctl enable-linger
```

## Install necessary software

```
sudo apt install -y pipewire-audio pulseaudio-utils bluez-tools python3-dbus wireplumber pipewire-pulse libspa-0.2-bluetooth
```
Some packages are already part of the distribution.

## Enable sound routing and bluetooth

```
systemctl --user enable --now pipewire pipewire-pulse wireplumber
systemctl enable --now bluetooth
```

## Configure Bluetooth

Setup Bluetooth to handle automatic connections gracefully and to enable A2DP (experimental feature)
```
sudo nano /etc/bluetooth/main.conf
```
Then change/enter parameter:
```
[General]
Name = Pi-BT-Stream
Class = 0x200414
DiscoverableTimeout = 0
PairableTimeout = 0
AutoEnable=true
Experimental = true
JustWorksRepairing = always
```

## Setup Service (agent) for autoconnect.
Setup the System so that it will accept any bluetooth connection request at any time without confirmation, using the python script from [fdanis-oss/pw_wp_bluetooth_rpi_speaker](https://github.com/fdanis-oss/pw_wp_bluetooth_rpi_speaker).

A general agent script can be found in the [Bluez Documentation](https://github.com/bluez/bluez/blob/master/test/simple-agent)
```
sudo nano /usr/local/bin/bt-agent.py
```
Paste the script:
```
#!/usr/bin/python3
# SPDX-License-Identifier: LGPL-2.1-or-later
# This Script comes from: https://github.com/fdanis-oss/pw_wp_bluetooth_rpi_speaker

import argparse
import dbus
import dbus.service
import dbus.mainloop.glib
from gi.repository import GLib

BUS_NAME = 'org.bluez'
AGENT_INTERFACE = 'org.bluez.Agent1'
AGENT_PATH = "/speaker/agent"

A2DP = '0000110d-0000-1000-8000-00805f9b34fb'
AVRCP = '0000110e-0000-1000-8000-00805f9b34fb'

bus = None


class Rejected(dbus.DBusException):
    _dbus_error_name = "org.bluez.Error.Rejected"


class Agent(dbus.service.Object):

    def __init__(self, bus, path, single_connection):
        self.exit_on_release = True
        self.remote_device = None

        dbus.service.Object.__init__(self, bus, path)

        if single_connection:
            bus.add_signal_receiver(self.signal_handler,
                                    bus_name='org.bluez',
                                    interface_keyword='org.freedesktop.DBus.Properties',
                                    member_keyword='PropertiesChanged',
                                    arg0='org.bluez.Device1',
                                    path_keyword='path'
                                    )

    def signal_handler(self, *args, **kwargs):
        path = kwargs['path']
        connected = None
        for i, arg in enumerate(args):
            if type(arg) == dbus.Dictionary and "Connected" in arg:
                connected = arg["Connected"]

        if connected == None:
            return

        if not self.remote_device and connected == True:
            self.remote_device = path
            print("{} connected".format(path))
        elif path == self.remote_device and connected == False:
            self.remote_device = None
            print("{} disconnected".format(path))

    def set_exit_on_release(self, exit_on_release):
        self.exit_on_release = exit_on_release

    @dbus.service.method(AGENT_INTERFACE,
                         in_signature="", out_signature="")
    def Release(self):
        print("Release")
        if self.exit_on_release:
            mainloop.quit()

    @dbus.service.method(AGENT_INTERFACE,
                         in_signature="os", out_signature="")
    def AuthorizeService(self, device, uuid):
        if self.remote_device and self.remote_device != device:
            print("%s try to connect while %s already connected" % (device, self.remote_device))
            raise Rejected("Connection rejected by user")

        # Always authorize A2DP and AVRCP connection
        if uuid in [A2DP, AVRCP]:
            print("AuthorizeService (%s, %s)" % (device, uuid))
            return
        else:
            print("Service rejected (%s, %s)" % (device, uuid))
        raise Rejected("Connection rejected by user")

    @dbus.service.method(AGENT_INTERFACE,
                         in_signature="", out_signature="")
    def Cancel(self):
        print("Cancel")


def start_speaker_agent():
    # By default Bluetooth adapter is not discoverable and there's
    # a 3 min timeout
    # Set it as always discoverable
    adapter = dbus.Interface(bus.get_object(BUS_NAME, "/org/bluez/hci0"),
                             "org.freedesktop.DBus.Properties")
    adapter.Set("org.bluez.Adapter1", "DiscoverableTimeout", dbus.UInt32(0))
    adapter.Set("org.bluez.Adapter1", "Discoverable", True)

    print("RPi speaker discoverable")

    # As the RPi speaker will not have any interface, create a pairing
    # agent with NoInputNoOutput capability
    obj = bus.get_object(BUS_NAME, "/org/bluez")
    manager = dbus.Interface(obj, "org.bluez.AgentManager1")
    manager.RegisterAgent(AGENT_PATH, "NoInputNoOutput")

    print("Agent registered")

    manager.RequestDefaultAgent(AGENT_PATH)


def nameownerchanged_handler(*args, **kwargs):
    if not args[1]:
        print('org.bluez appeared')
        start_speaker_agent()


if __name__ == '__main__':
    options = argparse.ArgumentParser(description="BlueZ Speaker Agent")
    options.add_argument("--single-connection", action='store_true', help="Allow only one connection at a time")
    args = options.parse_args()

    dbus.mainloop.glib.DBusGMainLoop(set_as_default=True)

    bus = dbus.SystemBus()

    agent = Agent(bus, AGENT_PATH, args.single_connection)
    agent.set_exit_on_release(False)

    bus.add_signal_receiver(nameownerchanged_handler,
                            signal_name='NameOwnerChanged',
                            dbus_interface='org.freedesktop.DBus',
                            path='/org/freedesktop/DBus',
                            interface_keyword='dbus_interface',
                            arg0='org.bluez')

    dbus_service = bus.get_object('org.freedesktop.DBus',
                                  '/org/freedesktop/DBus')
    dbus_dbus = dbus.Interface(dbus_service, 'org.freedesktop.DBus')
    if (dbus_dbus.NameHasOwner('org.bluez')):
        print('org.bluez already started')
        start_speaker_agent()

    mainloop = GLib.MainLoop()
    mainloop.run()
```

Make the script executable
```
sudo chmod +x /usr/local/bin/bt-agent.py
```
## Create a systemd service
For the agent script we create a systemd service so it will start and restart automaticaly.
```
sudo nano /etc/systemd/system/bt-auto-agent.service
```
contents: 
```
[Unit]
Description=BlueZ Auto Pairing Agent (NoInputNoOutput)
After=bluetooth.service
Requires=bluetooth.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /usr/local/bin/bt-agent.py
Restart=always
RestartSec=1
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```
Then properly register the new service with the system and enable and start it at the same time. 
```
sudo systemctl daemon-reload
sudo systemctl enable --now bt-auto-agent.service
```
## Connect an A2DP source
Connect and A2DP source like a phone, start to play some music and route it to the Pi Bluetooth sink.
## Stream the music via http
The A2DP sink stream shows up as `auto_null.monitor`. You can check with

```pactl list sources short```


Then start the stream:
```
ffmpeg -f pulse -i auto_null.monitor -ac 2 -ar 44100 -b:a 128k -f mp3 -listen 1 http://0.0.0.0:8000/
```
