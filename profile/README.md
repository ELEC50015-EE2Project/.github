# Double Pendulum FPGA Implementation

For each module/codebase, create a separate repository (within the project) and push your code there.

# Overall Architecture

<img width="2518" height="1274" alt="image" src="https://github.com/user-attachments/assets/8904dbd0-65dc-4a66-8e4c-acd61cafa4b5" />

**Optimisations Ideas:**

We don't even need the Round Round Arbiter and 2x Evicition Logic and Colour Map. Have the scheduler delay the start of the HLS IP, by not sending the state. Now there will only ever be one next state coming off both HLS IP at the same time, so both can be routed to 1 Eviction Logic block and so on. Throughput remains the same.

---

# AXI Architecture

## Primitives

### State Primitive (192 bits)

| Bits      | Width | Field    | Description               |
|-----------|-------|----------|---------------------------|
| [191:165] | 27b   | Padding  | Tied to zero              |
| [164]     | 1b    | Flipped? | Has the pendulum flipped? |
| [163:144] | 20b   | Address  | Pixel index (0–921,599)   |
| [143:128] | 16b   | Count    | RK4 steps taken so far    |
| [127:96]  | 32b   | ω₂       | Q16.16                    |
| [95:64]   | 32b   | ω₁       | Q16.16                    |
| [63:32]   | 32b   | θ₂       | Q16.16                    |
| [31:0]    | 32b   | θ₁       | Q16.16                    |

### Pixel Primitive (48 bits)

| Bits    | Width | Field   | Description  |
|---------|-------|---------|--------------|
| [47:44] | 4b    | Padding | Tied to zero |
| [43:24] | 20b   | Address | Pixel index  |
| [23:16] | 8b    | Red     | R channel    |
| [15:8]  | 8b    | Green   | G channel    |
| [7:0]   | 8b    | Blue    | B channel    |

---

# Modules

To edit an IP in Vivado, run this command in the tcl window:

```tcl
ipx::edit_ip_in_project -upgrade true -name edit_ip_project -directory C:/Users/.../ColourMap/ColourMap_1.0 C:/Users/.../ColourMap/ColourMap_1.0/component.xml
```

The generated skeleton code is quite verbose and full of nothing; the most important thing is the top level as that's where most of the logic can be written. The sub files aren't necessary, apart from the AXI Lite handler.

---

### IC Loader

| Direction | Interface             | Description                           |
|-----------|-----------------------|---------------------------------------|
| Input     | AXI4 Full Master Read | Reads IC table from DDR               |
| Input     | AXI4-Lite Slave       | Configuration registers (see below)   |
| Output    | AXI4-Stream Master    | Writes state tokens to New State FIFO |

**AXI-Lite Registers:**

| Register | Field   | Description                                                      |
|----------|---------|------------------------------------------------------------------|
| Reg 0    | ω₁      | Initial angular velocity, Q16.16                                 |
| Reg 1    | ω₂      | Initial angular velocity, Q16.16                                 |
| Reg 2    | Count   | Total pixel count                                                |
| Reg 3    | Base address | Base address of IC table in DDR — **write to trigger re-render** |

**Behaviour:**
1. If the New State FIFO is not full, fetch the next IC table entry from DDR. Each entry contains θ₁ and θ₂.
2. Combine the fetched θ₁, θ₂ with ω₁, ω₂ from AXI-Lite, `count = 0`, `address = entry index`, and `flipped = 0` to form a complete State Primitive.
3. Push the token onto the New State FIFO via AXI-Stream, then advance to the next entry.
4. On new parameters, restart from entry 0. Stop once all IC entries have been dispatched.

**Software:**
Write omega and pixel count at any time. To trigger a re-render, write to Reg 3 (base address) — it doesn't need to change value; the write event itself starts the re-render. This prevents updating other parameters from triggering a re-render before all parameters have been set.

> **Note:** The FIFO requests data from the IC Loader; it should request data when there are at least 8 free spaces in the FIFO, as that corresponds to the burst size configured in the IP.

---

### Scheduler

| Direction | Interface          | Description          |
|-----------|--------------------|----------------------|
| Input     | AXI4-Stream Slave  | New State FIFO       |
| Input     | AXI4-Stream Slave  | Executing State FIFO |
| Output    | AXI4-Stream Master | HLS core 0           |
| Output    | AXI4-Stream Master | HLS core 1           |

**Behaviour:**
1. Forwards state tokens to whichever HLS IP core asserts `TREADY`, indicating it is ready to accept a new token.
2. The Executing State FIFO has priority over the New State FIFO.
3. `TDATA` is passed through unmodified.

---

### HLS IP Core (×N)

| Direction | Interface          | Description                        |
|-----------|--------------------|------------------------------------|
| Input     | AXI4-Stream Slave  | State token from Scheduler         |
| Input     | AXI4-Lite Slave    | Configuration registers (see below)|
| Output    | AXI4-Stream Master | Updated state token                |

**AXI-Lite Registers:**

| Register | Field | Description                            |
|----------|-------|----------------------------------------|
| Reg 0    | m₁    | Mass of first arm, Q16.16              |
| Reg 1    | m₂    | Mass of second arm, Q16.16             |
| Reg 2    | L₁    | Length of first arm, Q16.16            |
| Reg 3    | L₂    | Length of second arm, Q16.16           |
| Reg 4    | g     | Gravitational acceleration, Q16.16     |
| Reg 5    | dt    | Integration timestep, Q16.16           |

**Behaviour:**
1. Performs one RK4 integration step on the incoming state, advancing (θ₁, θ₂, ω₁, ω₂) by dt.
2. Evaluates whether the second arm has completed a full rotation (flip detection).

---

### Eviction Logic (×2)

| Direction | Interface          | Description                        |
|-----------|--------------------|------------------------------------|
| Input     | AXI4-Stream Slave  | State token from HLS core          |
| Input     | AXI4-Lite Slave    | Configuration registers (see below)|
| Output    | AXI4-Stream Master | Finished state stream              |
| Output    | AXI4-Stream Master | Still-executing state stream       |

**AXI-Lite Registers:**

| Register | Field       | Description                                       |
|----------|-------------|---------------------------------------------------|
| Reg 0    | `max_count` | Evict token when count reaches this value         |

**Behaviour:**
1. If `flipped == 1` or `count == max_count`, evict the token — send it downstream on the Finished State stream.
2. Otherwise, increment `count` and return the token to the Executing State FIFO.

---

### Colour Map

| Direction | Interface          | Description                        |
|-----------|--------------------|------------------------------------|
| Input     | AXI4-Stream Slave  | Finished state token               |
| Input     | AXI4-Lite Slave    | Configuration registers (see below)|
| Output    | AXI4-Stream Master | Pixel Primitive                    |

**AXI-Lite Registers:**

| Register | Field       | Description                                  |
|----------|-------------|----------------------------------------------|
| Reg 0    | `max_count` | Upper bound for linear colour mapping        |

**Behaviour:**
1. Applies a linear mapping of `count` over the range `[0, max_count]` to an RGB colour value.
2. Packages the resulting RGB with the token's `address` field into a Pixel Primitive and forwards it downstream.

---

### Round Robin Arbiter

| Direction | Interface          | Description   |
|-----------|--------------------|---------------|
| Input     | AXI4-Stream Slave  | Input 0       |
| Input     | AXI4-Stream Slave  | Input 1       |
| Output    | AXI4-Stream Master | Merged output |

**Behaviour:**
1. If data is present on only one input, forward it immediately.
2. If data is present on both inputs simultaneously, forward one this cycle and buffer the other, then forward the buffered token next cycle.

---

### Pixel Writer

| Direction | Interface             | Description                          |
|-----------|-----------------------|--------------------------------------|
| Input     | AXI4-Stream Slave     | Pixel Primitives from upstream FIFO  |
| Input     | AXI4-Lite Slave       | Configuration registers (see below)  |
| Output    | AXI4 Full Master Write| Writes pixel data to DDR via HP port |

**AXI-Lite Registers:**

| Register | Field        | Description                           |
|----------|--------------|---------------------------------------|
| Reg 0    | Base address | Base address of frame buffer 0 in DDR |
| Reg 1    | Base address | Base address of frame buffer 1 in DDR |

**Behaviour:**
1. Reads Pixel Primitives from the input FIFO.
2. Writes each pixel's RGB data to the DDR address `frame_buffer_base + address`, where `address` is the pixel index carried in the Pixel Primitive.
3. Performs the above write for **both** frame buffers.

---

# Software

Here are a list of things you need to be aware of from a hardware side:

1. All the AXI-Lite registers need to be programmed for the modules above.
2. The controller sends data over SPI which is then read by the PYNQ board's hardware into a FIFO, and then drained by the ARM CPU. [Example](https://github.com/ELEC50015-EE2Project/SoftwareExamples/blob/main/spi.py)
3. To trigger a re-render, you have to write to the AXI-Lite register for the initial condition base table (IC Loader Reg 3). It doesn't even have to change the value stored in the register — just a write to the register starts the re-render. This prevents updating the parameters from triggering the re-render before all parameters have been set by the user. [Example](https://github.com/ELEC50015-EE2Project/SoftwareExamples/blob/main/ic_loader.py)
4. Under a somewhat unorthodox arrangement, part of the frame buffer is to be modified only by HW (the Pixel Writer IP) and part of it by SW for animation and general information display. This means both HW and SW can update the frame buffer(s) independently, which prevents SW from having to constantly pull pixels from hardware and manually write them to the frame buffer. The SW can then be free to focus on running animations for the display. Note that writing to the DDR automatically causes the frame buffer to be updated, as the VDMA automatically scans the same frame buffer location and outputs whatever is there.
