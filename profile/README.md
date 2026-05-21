# Double Pendulum FPGA Implementation

For each module/codebase, create a seperate repository (within the project) and push your code there. 

# AXI Architecture

## Primitives

### State Primitive (192 bits)

| Bits      | Width | Field    | Description                        |
|-----------|-------|----------|------------------------------------|
| [191:165] | 28b   | Padding  | Tied to zero                       |
| [164]     | 1b    | Flipped? | Has the pendulum flipped?          |
| [163:144] | 20b   | Address  | Pixel index (0–921,599)            |
| [143:128] | 16b   | Count    | RK4 steps taken so far             |
| [127:96]  | 32b   | ω₂       | Q16.16                             |
| [95:64]   | 32b   | ω₁       | Q16.16                             |
| [63:32]   | 32b   | θ₂       | Q16.16                             |
| [31:0]    | 32b   | θ₁       | Q16.16                             |

### Pixel Primitive (44 bits)

| Bits    | Width | Field   | Description     |
|---------|-------|---------|-----------------|
| [43:24] | 20b   | Address | Pixel index     |
| [23:16] | 8b    | Red     | R channel       |
| [15:8]  | 8b    | Green   | G channel       |
| [7:0]   | 8b    | Blue    | B channel       |

---

## Modules

### IC Loader

| Direction | Interface         | Description                        |
|-----------|-------------------|------------------------------------|
| Input     | AXI4 Full Master Read  | Reads IC table from DDR       |
| Output    | AXI4-Stream Master     | Writes state tokens to New State FIFO |

**AXI-Lite registers:**
- ω₁, ω₂ (initial angular velocities)
- Base address of IC table in DDR

**Behaviour:**
1. If the New State FIFO is not full, fetch the next IC table entry from DDR. Each entry contains θ₁ and θ₂.
2. Combine the fetched θ₁, θ₂ with ω₁, ω₂ from AXI-Lite, `count = 0`, `address = entry index`, and `flipped = 0` to form a complete State Primitive.
3. Push the token onto the New State FIFO via AXI-Stream, then advance to the next entry.
4. On new parameters, restart from entry 0. Stop once all IC entries have been dispatched.

---

### Scheduler

| Direction | Interface              | Description                        |
|-----------|------------------------|------------------------------------|
| Input     | AXI4-Stream Slave      | New State FIFO                     |
| Input     | AXI4-Stream Slave      | Executing State FIFO               |
| Output    | AXI4-Stream Master     | HLS core 0                         |
| Output    | AXI4-Stream Master     | HLS core 1                         |

**Behaviour:**
1. Forwards state tokens to whichever HLS IP core asserts `TREADY`, indicating it is ready to accept a new token.
2. The Executing State FIFO has priority over the New State FIFO.
3. `TDATA` is passed through unmodified.

---

### HLS IP Core (×N)

| Direction | Interface              | Description                        |
|-----------|------------------------|------------------------------------|
| Input     | AXI4-Stream Slave      | State token from Scheduler         |
| Output    | AXI4-Stream Master     | Updated state token                |
| Output    | Signal (1b)            | Flipped?                           |

**AXI-Lite registers:**
- m₁, m₂
- L₁, L₂
- g
- dt

**Behaviour:**
1. Performs one RK4 integration step on the incoming state, advancing (θ₁, θ₂, ω₁, ω₂) by dt.
2. Evaluates whether the second arm has completed a full rotation (flip detection).

---

### Eviction Logic (×2)

| Direction | Interface              | Description                        |
|-----------|------------------------|------------------------------------|
| Input     | AXI4-Stream Slave      | State token from HLS core          |
| Input     | Signal (1b)            | Flipped?                           |
| Output    | AXI4-Stream Master     | Finished state stream              |
| Output    | AXI4-Stream Master     | Still-executing state stream       |

**AXI-Lite registers:**
- `max_count`

**Behaviour:**
1. If `flipped == 1` or `count == max_count`, evict the token — send it downstream on the Finished State stream.
2. Otherwise, increment `count` and return the token to the Executing State FIFO.

---

### Colour Map

| Direction | Interface              | Description                        |
|-----------|------------------------|------------------------------------|
| Input     | AXI4-Stream Slave      | Finished state token               |
| Output    | AXI4-Stream Master     | Pixel Primitive                    |

**AXI-Lite registers:**
- `max_count`

**Behaviour:**
1. Applies a linear mapping of `count` over the range `[0, max_count]` to an RGB colour value.
2. Packages the resulting RGB with the token's `address` field into a Pixel Primitive and forwards it downstream.

---

### Round Robin Arbiter

| Direction | Interface              | Description                        |
|-----------|------------------------|------------------------------------|
| Input     | AXI4-Stream Slave      | Input 0                            |
| Input     | AXI4-Stream Slave      | Input 1                            |
| Output    | AXI4-Stream Master     | Merged output                      |

**Behaviour:**
1. If data is present on only one input, forward it immediately.
2. If data is present on both inputs simultaneously, forward one this cycle and buffer the other, then forward the buffered token next cycle.

---

### Frame Writer

| Direction | Interface              | Description                                  |
|-----------|------------------------|----------------------------------------------|
| Input     | AXI4-Stream Slave      | Pixel Primitives from upstream FIFO          |
| Output    | AXI4 Full Master Write | Writes pixel data to DDR via HP port         |

**AXI-Lite registers:**
- Base address of the frame buffer in DDR

**Behaviour:**
1. Reads Pixel Primitives from the input FIFO.
2. Writes each pixel's RGB data to the DDR address `frame_buffer_base + address`, where `address` is the pixel index carried in the Pixel Primitive.
