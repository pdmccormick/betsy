The Betsy LED display protocol
==============================

## Introduction

The Betsy display is composed of individually addressable modular tiles.  Each
tile is a combination of a 1 sq. ft. of high output LEDs plus a logic
controller with Ethernet connectivity. The controller speaks IPv6 and
understands a simple textual command format sent via UDP. Each tile operates
entirely independently but can be coordinated together for synchronous display
updates.


## IPv6 addressing

Each tile is addressed either by a unicast [link-local
address](https://en.wikipedia.org/wiki/Link-local_address#IPv6) with prefix of
`fe80::` (example `fe80::204:a3ff:fe1b:9324`) or the special [all-nodes
multicast address](https://en.wikipedia.org/wiki/Multicast_address#IPv6)
`ff02::1`.

Tiles respond to ICMP6 pings. For example, from a Linux host, assuming that
`eth0` is the interface connected to the same network segment as a tile (use
`ifconfig` to check for other possibilities):

```shell
$ ping6 -c 3 'fe80::204:a3ff:fe1b:9324%eth0'
PING fe80::204:a3ff:fe1b:9324%eth0(fe80::204:a3ff:fe1b:9324) 56 data bytes
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=1 ttl=64 time=0.225 ms
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=2 ttl=64 time=0.228 ms
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=3 ttl=64 time=0.244 ms

--- fe80::204:a3ff:fe1b:9324%eth0 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.225/0.232/0.244/0.015 ms
```

Or, using the all-nodes multicast address (note that all IPv6 enabled hosts
will respond to these ping requests):

```shell
$ ping6 -c 3 'ff02::1%eth0'
PING ff02::1%eth0(ff02::1) 56 data bytes
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=1 ttl=64 time=0.241 ms
... other responses ...
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=2 ttl=64 time=0.246 ms
... other responses ...
64 bytes from fe80::204:a3ff:fe1b:9324: icmp_seq=2 ttl=64 time=0.246 ms

--- ff02::1%eth0 ping statistics ---
3 packets transmitted, 3 received, +## duplicates, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.040/57.796/171.074/69.305 ms
```

## UDP protocol

Tiles listen on and respond from UDP port `48757`. The payload consists of one
or more 8-bit ASCII text commands, separated by semicolons. In the case of
sending or receiving binary payloads, all of the remainder of the packet after
the semicolon is considered to be the payload. Commands take the form of space
separated keywords and integers.

Using the `betsy_command` utility from the
[python-betsy](https://github.com/pdmccormick/python-betsy) package, here are
some simple commands and their responses from a tile:

```shell
$ ./betsy_command eth0 'ff02::1' 'uptime'
Sending: uptime
[fe80::204:a3ff:fe1b:9324%eth0] :: b'uptime bootldr 3244.370 ;'
Received 1 replies

$ ./betsy_command eth0 'ff02::1' 'uptime; uptime'
Sending: uptime; uptime
[fe80::204:a3ff:fe1b:9324%eth0] :: b'uptime bootldr 3251.730 ;uptime bootldr 3251.730 ;'
Received 1 replies

$ ./betsy_command eth0 'ff02::1' 'buildstamp bootldr'
Sending: buildstamp bootldr
[fe80::204:a3ff:fe1b:9324%eth0] :: b"buildstamp bootldr 'mode=bootldr model=itc_mse-0.2 dpc=t16-12-12v0.9 builder=pdm@sarpsborg commit=e69b619004e53b2c56956cecd57596ffbedb7718 ts=2016-08-06T05:26:59.310844Z';"
Received 1 replies
```

## Image data format

The Betsy class tiles include a DPC (_Display PCB Component_) consisting of
18x18 pixels (WxH), each with high output discrete red, green and blue LEDs.
Each colour channel is driven by a 12-bit constant current PWM driver IC. Each
tile controller maintains several frame buffers (currently 8, numbered from 0),
each consisting of little endian 16-bit per pixel-channel values (48-bit per
pixel) arranged left to right, top to bottom. Each buffer is thus `18*18*3*2`
or `1944` bytes in size. The high order 4 bits of each 16-bit channel value are
ignored.

Data is transmitted into the frame buffer using the `dpc data` command, and
framebuffers are pushed out to the DPC using the `dpc upload` command.

Since the frame buffer size of 1944 bytes is greater than the Ethernet MTU of
1500, typically two `dpc data` packets are used to transmit the entire frame:

> First command packet (assuming upload into framebuffer #1):
>
>       dpc data 1 0;...1024 bytes of PWM framebuffer data...
>
> Second command packet:
>
>       dpc data 1 1024;...remaining 920 bytes of framebuffer data...


## Command reference

### `uptime`

> Command packet:
>
>       uptime
>
> Example response (assuming tile is in bootloader mode):
>
>       uptime bootldr 3849.260 ;

Respond with the current operating mode and the number of seconds that the
controller has been running in that mode form. Example response:


### `reset`: reset into bootloader or firmware

Switch between the bootloader and firmware modes. DPC functionality is only
available from firmware mode.

> Command packet:
>
>       reset

Resets the controller into bootloader mode.

> Command packet:
>
>       reset bootldr

Another way of resetting the controller into bootloader mode.

> Command packet:
>
>       reset firmware

Resets the controller into firmware mode (only from bootloader mode.)


### `buildstamp`: identify the mode and build information about the bootloader/firmware image

> Command packet:
>
>       buildstamp
>
> Example response (assuming tile is in bootloader mode):
>
>       buildstamp bootldr 'mode=bootldr model=itc_mse-0.2 dpc=t16-12-12v0.9 builder=pdm@sarpsborg commit=e69b619004e53b2c56956cecd57596ffbedb7718 ts=2016-08-06T05:26:59.310844Z'

Returns the buildstamp for whatever the current mode is (bootloader or
firmware). The buildstamp is a formatted string indicating some build-time
details of the current image, such as when it was compiled and for which
controller variant.

> Command packet:
>
>       buildstamp bootldr

Returns the bootloader buildstamp.

> Command packet:
>
>       buildstamp firmware 

Returns the firmware buildstamp.


### `dpc` and `dpc!`

Controls the DPC (_Display PCB Component_) portion of the tile. Used to
transfer image data and refresh the display. Most `dpc` subcommands will return
some kind of acknowledgement response, but these can be silenced by using the
alternative `dpc!` command verb.

The `dpc` commands are only available when the tile is in firmware mode. In
bootloader mode, the tiles will respond with `error dpc bad-verb`.


#### `dpc info`

Reports DPC configuration details.

> Command packet:
>
>       dpc info
>
> Response:
>
>       dpc info bus_addrs=64 bus_bits=1024 gclk=1 gain=64 xhorz=18 yvert=18 num_addr=64 num_port=16 pwmframe_size=1944 num_pwmframes=8 ;


#### `dpc clear` and `dcp clear BUFFER`

Turns off all LEDs on the DPC. If _BUFFER_ is specified, then in addition to
turning off the LEDs that framebuffer is zeroed out and the min/max offset
indicators are reset to -1.


#### `dpc gclk 0/1/on/off`

Activates or deactivates the internal DPC clocking signal. If the GCLK is off,
you can still transmit and upload framebuffer data, but no LEDs will be
activated.

Note: Turning off the GCLK is a considerate way of testing transmission code
without flickering the display.


#### `dpc gain VALUE`

Sets the internal current gain of the LEDs.


#### `dpc data BUFFER OFFSET`

Copies the remaining payload bytes to the specified framebuffer buffer,
starting from some offset. Will set the min/max offset indicators as reported
by `dpc stats` and handled by `dpc nack`.


#### `dpc upload BUFFER`

Uploads the specified framebuffer to the DPC, updating the display. Resets the
min/max offset indicators for the framebuffer, but leaves the data itself
intact.

After copying DPC data to each node via its link-local unicast address,
transmit this command packet to the all-nodes mulitcast address `ff02::1` to
synchronously update the image across all tiles.


#### `dpc stats` and `dpc stats reset`

Returns some stats about the number of DPC commands received, and the current
min/max offset indicators for each framebuffer.


#### `dpc nack BUFFER MIN_OFFSET MAX_OFFSET`

To be documented. Only responds if the given framebuffer min/max offset
indicators are different. Can be used to confirm delivery of a complete
framebuferr worth of data.
