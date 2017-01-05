---
layout: post
title:  "CoreMIDI library for Rust"
date:   2017-01-04 21:30:00
categories: software
tags: rust midi osx coremidi
comments: True
---
# Introduction

[CoreMIDI](https://developer.apple.com/reference/coremidi) is a Mac OSX framework that provides APIs for communicating with [MIDI](https://www.midi.org/) (Musical Instrument Digital Interface) devices, including hardware keyboards and synthesizers. It has a C interface so it can be used by almost any programming language. On the other side there is [Rust](https://www.rust-lang.org/), a systems programming language that runs blazingly fast, prevents segfaults, and guarantees thread safety.

This blog post is to announce my first contribution to the open source Rust community with a library to interface CoreMIDI with Rust.

The initial goal with this library was to cover the basic functionality from the CoreMIDI framework and let the door open for contributions to fulfill the rest of the functionalities. By basic functionality I mean:

- Being able to create a client.
- Being able to send MIDI data either to output ports or through virtual sources.
- Being able to receive MIDI data either from input ports connected to sources or through virtual destinations.
- Enumerate system source and destination endpoints.

But the CoreMIDI framework includes much more things such as:

- Client notifications.
- Objects properties.
- Sysex data.
- Enumerate devices and entities.
- MIDI Setup.
- MIDI Thru connections.
- MIDI network connections.

My main motivation to get involved on this project, was the fact that I am building my own MIDI controllers with micro-controllers, and needed to interface them to my computer. Given that I work now with a Mac OSX laptop, I initially tried to use Swift and its official CoreMIDI binding, but soon I discovered that Swift is still in its very early days regarding package management, building tooling, and open source community. Then I went to see what Rust could provide me in this field, and it was quite surprising to find some work around CoreMIDI. Unfortunately no one provided a perfect solution for my project, but it was a very good starting point for me to start working on my own library.

These are the libraries that existed when I decided to work on my own:

- [portmidi-rs](https://github.com/musitdev/portmidi-rs): It is a Rust library for [PortMIDI](http://portmedia.sourceforge.net/portmidi/) (written in C), which supports Linux, Mac, and Windows. It was almost perfect for me, except that I was not able to find how to create virtual sources (which was one of the main requirements for my controllers).
- [midir](https://github.com/Boddlnagg/midir): It aims to be a multi-platform MIDI library written in Rust, and it supported virtual ports, but there was no support for Mac OS yet. This is why I decided to create this CoreMIDI library that could be used to fill this gap by supporting the implementation of a Mac OS backend.
- [core-midi](https://github.com/dtwood/core-midi-rs): The aim of this library was exactly the same as mine. In fact, I initially thought that I could use it for my project, but soon I realised that it was quite incomplete, its author was not that responsive, and it required quite some work and love. To be fair I have to say that its source code was very important for me to understand how to interface callbacks from C with Rust.
- [coremidi](https://github.com/jonas-k/coremidi-sys). Then I found this other low-level bindings for CoreMIDI that looked complete, with its author more responsive. So I decided to build my own library on top of it. A part from some bugs that I am contributing to fix, it has been critical for the development of the library.

This is not the first time I write Rust (see my work in slow progress [herosynth](https://github.com/chris-zen/hero-synth) for my first Rust code started some years ago), but it is actually my first time getting dirty with the dark side of Rust, the **unsafe** territory. Hey, don't get me wrong, Rust main focus is on safety, and it is doing a great job there, but it also provides the tools to interface with unsafe libraries, as it is the case of the ones written in C. Writing this library has been a great experience to better understand the language and its constraints regarding safety. Hopefully, it won't be necessary to go to that level in the future because C will be something from the past ;-)

# Concepts

The following UML diagram depicts the main concepts available in CoreMIDI and how do they relate to each other according to my understanding from reading the API, and what this library exposes:

![Main UML Diagram](http://yuml.me/71202265.svg)
<!-- http://yuml.me/edit/71202265 -->

In the previous diagram I have omitted that several objects inherit from `Object`, which exposes several properties that can be introspected.

![Object UML Diagrams](http://yuml.me/97122448.svg)
<!-- http://yuml.me/edit/97122448 -->

This is what the official CoreMIDI documentation says:

*MIDI drivers own and control MIDI devices, which include such things as USB interfaces, MIDI keyboards, and so on. A device is defined as a physical object that would be represented by a single icon if there were a graphical view of the studio.*

*A MIDI device may have multiple logically distinct sub-components. For example, one device may encompass a MIDI synthesizer and a pair of MIDI ports, both addressable via a USB port. Each such element of a device is called a MIDI entity.*

*A MIDI entity can have any number of MIDI endpoints, each of which is a source or destination of a 16-channel MIDI stream. By grouping a device’s endpoints into entities, the system has enough information for an application to make reasonable default assumptions about how to communicate in a bi-directional manner with each entity, as is necessary in MIDI librarian applications.*

*Core MIDI attaches a collection of properties to each object it manages. Some properties are dynamic characteristics of a device—such as MIDI receive channel and system-exclusive IDs. Other properties are a matter of user preference—such as choice of icon, and whether or not the device should appear in lists of possible controllers. Still other properties are static and could be looked up in a database, using the device’s manufacturer and model names as a key.*

This library tries to follow those concepts as closely as possible, while feeling Rust idiomatic. The only addition I have done to this model is the `PacketBuffer`, which allows to build `PacketList`s. It dereferences to a `PacketList` so it can be used wherever a `PacketList` reference is needed.

![Packets UML Diagram](http://yuml.me/3d6d3cdd.svg)
<!-- http://yuml.me/edit/3d6d3cdd -->

# Installation

The library is published into [crates.io](https://crates.io/crates/coremidi), so it can be used by adding the following lines into your `Cargo.toml` file (but remember to update the version number accordingly [![Crates.io](https://img.shields.io/crates/v/coremidi.svg)](https://crates.io/crates/coremidi)):

{% highlight toml %}
[dependencies]
coremidi = "^0.0.2"
{% endhighlight %}

If you prefer to live in the edge ;-) you can use the master branch by including this instead:

{% highlight toml %}
[dependencies]
coremidi = { git = "https://github.com/chris-zen/coremidi", branch="master" }
{% endhighlight %}

To play with the source code yourself you can clone the repo and build the code and documentation with the following commands:

{% highlight sh %}
git clone https://github.com/chris-zen/coremidi.git
cd coremidi
cargo build
cargo test
cargo doc
open target/doc/coremidi/index.html
{% endhighlight %}

# Examples

The [documentation for the master branch](https://chris-zen.github.io/coremidi/coremidi/) is published automatically from Travis every time there is a push to master.

You can also find [several examples](https://github.com/chris-zen/coremidi/tree/master/examples) in the Github repository under the `examples` folder.

I know people is lazy and want to see code sooner than later, or at least this is the case for me ;-) so I am going to put here some code, even if it will most probably get out of date very soon.

You need a few lines to create a client, get an output port and send some MIDI data to the first endpoint available in the system (see [endpoints.rs](https://github.com/chris-zen/coremidi/blob/master/examples/send.rs) for further details and updated code):

{% highlight rust %}
extern crate coremidi;

use std::time::Duration;
use std::thread;

fn main() {
  let destination = coremidi::Destination::from_index(0);
  println!("Destination display name: {}", destination.display_name().unwrap());

  let client = coremidi::Client::new("example-client").unwrap();
  let output_port = client.output_port("example-port").unwrap();

  let note_on = coremidi::PacketBuffer::from_data(0, vec![0x90, 0x40, 0x7f]);
  let note_off = coremidi::PacketBuffer::from_data(0, vec![0x80, 0x40, 0x7f]);

  output_port.send(&destination, &note_on).unwrap();
  thread::sleep(Duration::from_millis(1000));
  output_port.send(&destination, &note_off).unwrap();
}
{% endhighlight %}

Every call that may fail returns a `Result<T, OSStatus>` so it needs to be either unwrapped or matched. As the error type, I decided to simply forward the `core_foundation_sys::base::OSStaus` code returned by the CoreMIDI API, but it may change in the future, not sure yet.

If you need to enumerate either the sources or the destinations available in the system you can use the following code (see [endpoints.rs](https://github.com/chris-zen/coremidi/blob/master/examples/endpoints.rs) for further details and updated code):

{% highlight rust %}
extern crate coremidi;

fn main() {
    println!("System destinations:");

    for (i, destination) in coremidi::Destinations.into_iter().enumerate() {
        let display_name = destination.display_name.unwrap();
        println!("[{}] {}", i, display_name);
    }

    println!("\nSystem sources:");

    for (i, source) in coremidi::Sources.into_iter().enumerate() {
        let display_name = source.display_name.unwrap();
        println!("[{}] {}", i, display_name);
    }
}
{% endhighlight %}

Finally, you can also receive MIDI messages very easily from the first available source in the system with (see [endpoints.rs](https://github.com/chris-zen/coremidi/blob/master/examples/receive.rs) for further details and updated code):

{% highlight rust %}
extern crate coremidi;

fn main() {
    let source = coremidi::Source::from_index(0);
    println!("Source display name: {}", source.display_name().unwrap());

    let client = coremidi::Client::new("example-client").unwrap();

    let callback = |packet_list: coremidi::PacketList| {
        println!("{}", packet_list);
    };

    let input_port = client.input_port("example-port", callback).unwrap();
    input_port.connect_source(&source).unwrap();

    let mut input_line = String::new();
    println!("Press [Intro] to finish ...");
    std::io::stdin().read_line(&mut input_line).ok().expect("Failed to read line");
}
{% endhighlight %}

A `PacketList` can be printed because it implements the traits Display and Debug.

There are a couple of examples to show how to send/receive MIDI messages by using virtual sources/destinations instead of output/input ports. I will skip them here but I encourage you to take a look into the examples [virtual-source.rs](https://github.com/chris-zen/coremidi/blob/master/examples/virtual-source.rs) and [virtual-destination.rs](https://github.com/chris-zen/coremidi/blob/master/examples/virtual-destination.rs) to see how easy it is.

# Conclusion

CoreMIDI is a very well designed library for Mac OS that allows communication with physical/virtual musical devices/instruments using the MIDI protocol. But it is interfaced following the C conventions (full of pointers, and unsafe by nature). This Rust library abstracts out the unsafe part of it, and allows users to focus on its services using idiomatic and safe Rust language.

This is the first time I write unsafe code in Rust, but it has allowed me to learn quite a lot about the language, and specially about the rules that govern its safety. Another important learning has been about how 'life after inheritance' is also possible. I still remember how much I missed inheritance the first time I started to code in Rust. But you can get similar results by using composition and `Deref`. The truth is that the trait system in Rust is really powerful, and the more I code in Rust, the more comfortable I feel with it. Not to mention, the great community that is being built around it.

I hope to find quite some collaboration from people interested in using it, and I am open to contributions to fill the gaps for unimplemented features. If I have enough time I would also like to contribute to [midir](https://github.com/Boddlnagg/midir) to provide a CoreMIDI backend so it can support Mac OS (but given how few spare time I usually have, I would also be glad to see other people to do it using my library ;-).
