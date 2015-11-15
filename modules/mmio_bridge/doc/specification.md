# Introduction

This document specifies the interface of the Memory-Mapped I/O Bridge
module. This bridge can be used by debug modules to abstract from the
Debug Packet format and use a simple memory-mapped I/O interface
instead. The debug functionality is then implemented as a slave module
to the bridge. Beside the ease of implementation is also allows the
actual debug functionality to be bridged to some other interconnect.

## License

This work is licensed under the Creative Commons
Attribution-ShareAlike 4.0 International License. To view a copy of
this license, visit
[http://creativecommons.org/licenses/by-sa/4.0/](http://creativecommons.org/licenses/by-sa/4.0/)
or send a letter to Creative Commons, PO Box 1866, Mountain View, CA
94042, USA.

You are free to share and adapt this work for any purpose as long as
you follow the following terms: (i) Attribution: You must give
appropriate credit and indicate if changes were made, (ii) ShareAlike:
If you modify or derive from this work, you must distribute it under
the same license as the original.

## Authors

Stefan Wallentowitz

Fill in your name here!

# Memory-Mapped Interface

The memory-mapped interface is a simple interface:

 Signal | Driver | Width | Description
 ------ | ------ | ----- | -----------
 `enable` | Master | 1 | Perform a transfer
 `addr` | Master | 16 | Address of the memory-mapped transfer.
 `data_in` | Master | 16 | Data to be written on write transfer
 `write` | Master | 1 | Signal a write transfer
 `mode` | Master | 2 | Mode of access
 `ack` | Slave | 1 | Acknowledge the transfer
 `data_out` | Slave | 16 | Read data
 `irq` | Slave | `IRQ_WIDTH` | Interrupt to start unsolicited transfer

The slave can acknowledge with the `ack` signal in the same cycle to
allow for high-speed interfacing, acknowledge in the next cycle or
insert arbitrarily long wait states. The `enable` signal gets low
after a synchronous `ack`.

`mode` can take the following values:

 `mode` | Description
 ------ | -----------
 `00` | Single-word access.
 `01` | reserved
 `10` | Incremental burst
 `11` | Same-address burst (FIFO-like)

The `irq` signal triggers the MMIO-bridge to read from an address as
described below and send a trace event to the host or configured
destination module. The irq is not a single bit event, but the signal
is set to the size of the trace event to transmit (limited by
`IRQ_WIDTH`). The slave module can determine success of the trace
event with the read sequence but does not have guarantees about the
time between an interrupt and the read sequence. The read sequence can
also contain pause states. Finally, the `irq` signal can be
de-asserted (set to `0x0` during any time between the start of the
first read (`enable=1 && addr=MMI_IRQ_CAUSE`) and the last
acknowledge. After this it must be de-asserted otherwise another trace
event is generated.

# Memory Map

The following map applies to the interface.

 Address Range | Description
 ------------- | -----------
 `0x0000` - `0x01ff` | Control and Status
 `0x0200` | Interrupt Cause
 `0x0201` - `0x02ff` | Reserved
 `0x0300` - `0xffff` | Slave module address space

## Control and Status

The slave module must handle the following control and status requests
properly, that are elaborated in the following:

 Address | Key | Description
 ------- | --- | -----------
 `0x0200` | `MMI_IRQ_CAUSE` | Interrupt Cause

### Interrupt Cause

On occurence of an interrupt, a trace event is generated and sent to
the host or configured destination. The value of the trace event is
determined by reading from this 16 bit register and the host or
destination module can take appropriate action.

Depending on the size signaled by the interrupt signal (see above)
`irq` count words are read from the address (same-address burst).

# Parameters

 Key | Width | Description
 --- | ----- | -----------
 `MOD_ID` | 16 | Module Identifier
 `MOD_VERSION` | 8 | Module Version
 `MOD_VENDOR` | 16 | Module Vendor (optional)


## Module Identifier (`MOD_ID`)

The module identifier can take two forms. If bit 15 is `0` it is an
Open SoC Debug module with a unique identifier in the bits `14:0`. If
bit 15 is `1` then the same bits decode a unique module version but of
a module vendor as defined by `MOD_VENDOR`.

## Module Vendor (`MOD_VENDOR`)

The optional vendor identifier (16 bit), that is only used if bit 15
of `MOD_ID` is used. For the use of a valid vendor identifier, please
refer to the Open SoC Debug specification.

## Module Version (`MOD_VERSION`)

An 8 bit version number of this module.