# Double Pendulum FPGA Flip Time Map Renderer

<p align="center">
<img width="720" height="720" alt="flipmap_720x720_dt0 005_mc4000_g9 81_w0-0_m1-1_L1-1_th1-3 14159-3 14159_th2-3 14159-3 14159" src="https://github.com/user-attachments/assets/afd338a4-f426-4194-bd17-d0ee2fd780dc" /><br/>
<em>Full flip time map: 720×720, dt = 0.005, max_count = 4000, g = 9.81, ω₁ = ω₂ = 0, m₁ = m₂ = 1, L₁ = L₂ = 1, θ₁ ∈ [−π, π], θ₂ ∈ [−π, π]</em>
</p>

## Overview

The double pendulum is a classic example of a simple mechanical system that
exhibits chaotic behaviour: two arms, two masses, and it's already impossible
to predict long-term motion from nearby starting conditions. Small changes in
initial angle produce wildly different trajectories.

This project exploits that sensitivity to turn the double pendulum into an
image generator. Each pixel in the output frame corresponds to a unique pair
of initial angles $(\theta_1, \theta_2)$. For every pixel, the system
integrates the double pendulum's equations of motion forward in time and
records how long it takes for the second arm to complete a full rotation (a
"flip"). That time is mapped to a colour, so the final image is a fractal-like
map of the system's chaotic structure.

The equations of motion (from the Lagrangian of the two-mass system) are:

$$
\dot\theta_1 = \omega_1, \qquad \dot\theta_2 = \omega_2
$$

$$
\dot\omega_1 = f_1(\theta_1, \theta_2, \omega_1, \omega_2), \qquad
\dot\omega_2 = f_2(\theta_1, \theta_2, \omega_1, \omega_2)
$$

where $f_1, f_2$ are the standard (nonlinear, coupled) double-pendulum angular
accelerations. These are integrated numerically using 4th-order Runge–Kutta:

$$
y_{n+1} = y_n + \frac{h}{6}\left(k_1 + 2k_2 + 2k_3 + k_4\right)
$$

Because every pixel is an independent integration, the problem is
embarrassingly parallel — which is what makes it a good fit for hardware
acceleration. Rather than running one CPU thread per pixel, the project
implements the RK4 integration loop directly in FPGA logic (via Vivado
HLS), so thousands of pixel-integrations can be in flight simultaneously,
streamed through a pipeline of custom AXI-connected IP blocks on a PYNQ
board.

<p align="center">
<img width="720" height="720" alt="flipmap_720x720_dt0 01_mc4000_g9 814_w0-0_m0 99-1_L0 99-1_th1-1 00933--0 633562_th22 02303-2 52114" src="https://github.com/user-attachments/assets/fe195719-4519-41a1-a216-c5a4c8ff0e6e" /><br/>
<em>Flip time map: 720×720, dt = 0.01, max_count = 4000, g = 9.814, ω₁ = ω₂ = 0, m₁ = 0.99, m₂ = 1, L₁ = 0.99, L₂ = 1, θ₁ ∈ [−1.00933, −0.633562], θ₂ ∈ [2.02303, 2.52114] (zoomed region)</em>
</p>

Each pixel's colour encodes how many RK4 steps its corresponding initial
condition took before the pendulum flipped, revealing the fractal boundary
between "fast flip" and "slow/no flip" regions of initial-condition space.

---

## AXI Architecture — Overview

<p align="center">
<img width="2518" height="1274" alt="architecture" src="https://github.com/user-attachments/assets/8904dbd0-65dc-4a66-8e4c-acd61cafa4b5" />
</p>

The design is a pipeline of AXI-Stream-connected IP blocks (AXI-Lite for
configuration, AXI4 Full for DDR access at the boundaries). A **State** token
(angles, angular velocities, iteration count, pixel address, flip flag)
circulates around the loop until it's finished, at which point it's converted
into a **Pixel** token (address + RGB) for writing to the frame buffer.

- **IC Loader** — streams each pixel's initial angles out of DDR as a new State token.
- **Scheduler** — hands tokens to whichever HLS core is free, prioritising in-progress states over new ones.
- **HLS IP Cores** — run one RK4 integration step per pass and check for a "flip".
- **Eviction Logic** — retires a token once it's flipped or hit the step limit; otherwise loops it back for another step.
- **Colour Map** — turns the final step count into an RGB colour.
- **Round Robin Arbiter** — merges the parallel pipeline outputs into one stream.
- **Pixel Writer** — writes each pixel to the frame buffer(s) in DDR, which the VDMA scans out to the display.
