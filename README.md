# Raspaudio
Turn a Raspberry Pi Zero 2 W into a **Bluetooth A2DP receiver that rebroadcasts the audio as an HTTP MP3 stream**.

I own an old internet radio receiver that has neither Bluetooth nor a line in. In order to play music from my phone I wanted a device that can act as a Bluetooth audio sink and forward the stream via http.
I am using a Raspberry Pi Zero 2 W with the latest Raspberry Pi OS (13.2). 

This project can be used for:

- Streaming phone audio into a local network
- Bluetooth audio bridge for legacy audio systems
- Remote audio monitoring

## Remarks
- The device needs a working network/wifi connection with an assigned IPv4 address. 
- The setup is done as the 'pi' default user, thus the 'sudo' commands.
- Some services (Bluetooth, Bluetooth-agent, ffmpeg) are set up as system services. The audio routing is done by the pi user.
- The project is in a 'works for me' state and has no regards of any other scripts or services on the machine.
- The device accepts any Bluetooth connection request automatically. Any nearby device can pair and connect.
- This was a days (on sick leave) work. There may be any kind of redundancies and odd detours. Any comments are welcome.

## Known Bugs
- The 'Name' in '/etc/bluetooth/main.conf' is not used, but the machine name.
- Sometimes you have to restart the system, re-pair or reconnect device for A2DP to work.

## To Dos
- When there is no Bluetooth audio stream present, play some default sound to allow an easy check if the http stream works

## Maybe's
- Turn this documentation into a setup_script.sh
- Move streaming from ffmpeg to icecast/darkice
- Make the device discoverable only a limited time after power on
- Make a minimal web interface for managing devices (discoverable on/off, remove paired devices)
- Harden the setup (readonly FS)
- Make a container version

## Base Installation

Start with a Raspberry Pi Os 32 Bit (with Desktop) Version 13.2 (Trixie).

The GUI won't be needed but the lite version might have issues with Bluetooth and/or A2DP.



## Setup the Pi for auto-login, disable GUI

Start raspi-config. Disable the GUI, enable auto-login and disable wifi energy saving.

To keep the session up any time:
```
sudo loginctl enable-linger
```

## Install necessary software

```
sudo apt install -y pipewire-audio pulseaudio-utils bluez-tools python3-dbus python3-gi wireplumber pipewire-pulse libspa-0.2-bluetooth
```
Some packages are already part of the distribution used here, some packages used in the setup may be missing on other distributions.

## Enable sound routing and bluetooth

```
systemctl --user enable --now pipewire pipewire-pulse wireplumber

systemctl enable --now bluetooth
```

## Configure Bluetooth

Setup Bluetooth to handle automatic connections gracefully and enable A2DP (experimental feature).
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
Setup the System so that it will accept any Bluetooth connection request at any time without confirmation, using the python script from [fdanis-oss/pw_wp_bluetooth_rpi_speaker](https://github.com/fdanis-oss/pw_wp_bluetooth_rpi_speaker).

For reference, a general agent script can be found in the [Bluez Documentation](https://github.com/bluez/bluez/blob/master/test/simple-agent).
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

Make the script executable:
```
sudo chmod +x /usr/local/bin/bt-agent.py
```
## Create a systemd connect service (agent)
For the agent script we create a systemd service so it will start and restart automaticaly.
```
sudo nano /etc/systemd/system/bt-auto-agent.service
```
paste contents: 
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
Then properly register the new service with the system and enable/start it. 
```
sudo systemctl daemon-reload
sudo systemctl enable --now bt-auto-agent.service
```
## Create a systemd streaming service
For the http mp3 stream script we also create a systemd service so it will start and restart automaticaly.
```
sudo nano /etc/systemd/system/httpstream.service
```
paste contents: 
```
[Unit]
Description=Bluetooth HTTP MP3 Stream
After=network.target sound.target

[Service]
ExecStart=/usr/bin/ffmpeg -f pulse -i auto_null.monitor -ac 2 -ar 44100 -b:a >
Restart=always
RestartSec=2
User=pi
Environment=PULSE_SERVER=unix:/run/user/1000/pulse/native

[Install]
WantedBy=multi-user.target
```

Then properly register the new service with the system and enable/start it. 

```
sudo systemctl daemon-reload
sudo systemctl enable --now httpstream.service
```

## Connect an A2DP source
Connect and A2DP source like a phone, start to play some music and route it to the Pi Bluetooth sink.



## Debugging Info / Troubleshooting
The A2DP sink stream shows up as `auto_null.monitor`. You can check the available sinks using

```pactl list sources short```

Check if Bluetooth audio is detected:

```wpctl status```

## Feedback

If you have questions, ideas, or improvements, feel free to:

- open an **Issue**
- start a **Discussion**
- submit a **Pull Request**

Feedback is very welcome.
