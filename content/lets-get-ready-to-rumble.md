---
title: "Let's get ready to rumble!"
description: "The future is now. Programming remote contolled vibrating devices."
date: 2020-01-25
---

Or: First steps in sending commands from Linux to a BLE device using Rust.

A while back I stumbled across the [buttplug.io](https://buttplug.io) project.
They aim to create an open sex toy control protocol specification and reference
implementations. They support various types of sex toys and other vibrating
peripherals, such as video game controller. Go check it out if you haven't done
so already!

So, a few days back I bought a Bluetooth Low Energy (BLE) capable toy.
A [Pivot](https://www.we-vibe.com/pivot) by We-Vibe.

Since there is an ongoing effort to create a Rust reference implementation
[buttplug-rs](https://github.com/buttplugio/buttplug-rs) and I want to learn
Rust, I decided to start by first creating a minimal PoC and later see what
else is possible with the framework. At the time of writing the buttplug-rs
did not seem to have support for my device.

The mechanism to connect to BLE devices from buttplug-rs is based on
[rumble](https://github.com/mwylde/rumble), a Rust library abstraction for the
BlueZ socket interface.

The rumble documentation has a simple usage example that shows how to control a
BLE light bulb. In the following code I adapted the example to send a "start
vibrating" commands to my device instead of "lights on" to a light bulb.

```rust
extern crate rumble;

use rumble::api::{Central, Peripheral, UUID};
use rumble::bluez::manager::Manager;
use std::thread;
use std::time::Duration;

fn main() {
    let manager = Manager::new().unwrap();

    // get the first bluetooth adapter
    let adapters = manager.adapters().unwrap();
    let mut adapter = adapters.into_iter().nth(0).unwrap();

    // reset the adapter -- clears out any errant state
    adapter = manager.down(&adapter).unwrap();
    adapter = manager.up(&adapter).unwrap();

    // connect to the adapter
    let central = adapter.connect().unwrap();

    // start scanning for devices
    central.start_scan().expect("start scan failed");
    // instead of waiting, you can use central.on_event to be notified of
    // new devices
    thread::sleep(Duration::from_secs(2));

    // find the device we're interested in
    let device = central
        .peripherals()
        .into_iter()
        .find(|p| {
            p.properties()
                .local_name
                .iter()
                .any(|name| name.contains("Pivot"))
        })
        .expect("could not find device..");

    println!("device found.");

    // connect to the device
    device.connect().expect("unable to connect");

    // discover characteristics
    device
        .discover_characteristics()
        .expect("could not discover device");

    // https://github.com/buttplugio/buttplug-device-config contains the UUIDs we use here.
    // The UUID byteorder is reversed in the definition below.
    let tx_uuid = UUID::B128([
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xB0, 0x00, 0x40, 0x51, 0x04, 0x00, 0xC0, 0x00,
        0xF0,
    ]);
    // find the characteristic we want
    let tx_char = device
        .characteristics()
        .into_iter()
        .find(|c| c.uuid == tx_uuid)
        .expect("could not find tx characteristic");

    // commands based on https://github.com/metafetish/wejibe-py
    let oncmd = vec![0x0f, 0x05, 0x00, 0xbc, 0x00, 0x00, 0x00, 0x00];
    let offcmd = vec![0x0f, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00];

    device.command(&tx_char, &oncmd).expect("unable to send..");
    println!("on");
    thread::sleep(Duration::from_millis(5000));
    device.command(&tx_char, &offcmd).expect("unable to send..");
    println!("off");
}
```

Having a small program like this will probably be useful to quickly test out
new commands on various devices without having to have a working buttplug-rs
setup, yet.

For information on device specific protocols see
[STPIHKAL](https://stpihkal.docs.buttplug.io/).
