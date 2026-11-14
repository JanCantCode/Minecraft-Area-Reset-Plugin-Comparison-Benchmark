# Benchmark comparison of various minecraft arena/area reset plugins

Minecraft arena/area reset plugins let server owners save a region and restore it after players modify or destroy it. This benchmark compares the CPU, memory and network costs of restoring damaged terrain with 11 implementations.

## Scenarios

Both scenarios used an 8 × 8 chunk natural landscape we picked randomly in the world, from Y=−64 through Y=319. A bot was online at the arena centre during every reset to ensure that client updates and network traffic would be sent.

**Fixed partial damage.** We removed a 40 × 40 × 64 block cuboid from the centre of the arena before each reset. The damage footprint covered 16 of the plot’s 64 chunks. 

**TNT destruction.** A central 20 × 20 × 20 cube of 8,000 TNT blocks was detonated before resetting. This destroyed large parts of the surface of the area and went far under the surface in some locations.

We picked these scenarios because partial destruction exposes additional optimization opportunities. Full destruction still allows optimizations in compression, packet selection, copying, lighting and avoiding unnecessary work. With partial destruction, plugins can also decide which sections need restoring, how to write them, which packets to use and when no update needs to be sent.
So in a sense, this benchmark would reflect a scenario like the plugins being used to reset an area around a Spawn where PVP happens, or in more general terms a PVP area, like it happens in crystal pvp, and less something like basic schematic pasting. 

## Methodology

- Paper 26.2 build 135, Java 25.0.2, a 3 GiB maximum server heap and four pinned physical CPU cores on an otherwise idle laptop CPU (13th gen intel i5-13500H)
- We ran each of the reset plugins on the same scenarios mentioned above with all other variables staying constant.
- We created the arenas beforehand, hence there is no impact from this.
- In order to do a fair comparison, we ran each plugin twice before doing the measurement as a warmup, for the measurement we did 5 runs and used the mean values between them.

## Metric definitions (For all of these; lower is better)

- **CPU usage (core-s):** JVM CPU time summed across cores. This is the usage across world ticking, reset plugin logic and netty threads.
- **Network sent (MiB):** The amount of data that was sent to the bot that was online. __this is after compression__
- **Heap allocated (MiB):** How much memory was allocated in total during the reset. We did our best to ensure this value is not influenced by background work from the server.
- **Peak network rate (MiB/s):** The highest network traffic spike we measured.
- **Worst tick (ms):** median of each trial’s longest server tick.

## Fixed partial-damage reset

| Implementation | CPU usage (core-s) | Network sent (MiB) | Heap allocated (MiB) | Peak network rate (MiB/s) | Worst tick (ms) |
|---|---:|---:|---:|---:|---:|
| AsyncArenasPlus | 0.234 | 0.196 | 5.30 | 0.609 | 1.74 |
| FastArena | 0.406 | 0.277 | 98.84 | 1.842 | 6.76 |
| AsyncArenas | 0.734 | 0.592 | 121.93 | 4.149 | 5.03 |
| Blossom | 0.734 | 0.596 | 121.40 | 4.721 | 4.89 |
| FastAsyncWorldEdit (FAWE) | 0.781 | 0.592 | 121.77 | 4.906 | 4.59 |
| NanoArenas | 0.781 | 0.595 | 122.01 | 3.508 | 4.97 |
| Arena Regeneration | 0.844 | 0.596 | 122.43 | 3.847 | 5.45 |
| AreaResetterPro | 0.953 | 0.593 | 141.34 | 4.170 | 5.89 |
| PerfAreaReset | 1.141 | 0.595 | 126.01 | 4.320 | 4.95 |
| AreaReloader-FAWE | 3.609 | 5.877 | 896.37 | 2.589 | 23.53 |
| WorldEdit | 22.766 | 11.518 | 5509.60 | 28.861 | 9960.35 |

## TNT reset

| Implementation | CPU usage (core-s) | Network sent (MiB) | Heap allocated (MiB) | Peak network rate (MiB/s) | Worst tick (ms) |
|---|---:|---:|---:|---:|---:|
| AsyncArenasPlus | 0.359 | 0.430 | 14.84 | 0.271 | 2.76 |
| FastArena | 0.438 | 0.299 | 102.16 | 1.569 | 6.20 |
| Blossom | 0.703 | 0.597 | 125.35 | 3.873 | 6.02 |
| FastAsyncWorldEdit (FAWE) | 0.828 | 0.594 | 126.89 | 4.719 | 7.39 |
| Arena Regeneration | 0.891 | 0.596 | 125.79 | 2.975 | 5.40 |
| NanoArenas | 0.953 | 0.597 | 124.81 | 3.584 | 5.46 |
| AsyncArenas | 0.969 | 0.594 | 125.64 | 4.362 | 5.35 |
| AreaResetterPro | 1.125 | 0.597 | 147.10 | 4.524 | 6.13 |
| PerfAreaReset | 1.297 | 0.597 | 135.05 | 3.865 | 5.06 |
| AreaReloader-FAWE | 3.625 | 5.545 | 890.56 | 2.567 | 20.27 |
| WorldEdit | 19.828 | 11.520 | 5502.73 | 29.728 | 9500.62 |

## Interpretation limits
We limited this benchmark to plugins that mainly aim to reset large parts of the arena. For instance, implementations that reset arenas by placing only a few blocks per tick on a budget were excluded since they result in a fundamentally different reset "experience".
In addition, AsyncArenas and AsyncArenasPlus are the only paid plugins in the list, since these are the only ones I have access to. If you own a license / are a developer of another paid alternative, I would love to put it to the test as well, please reach out!

Also obviously this benchmark only looked at resetting an area but makes no claims about general superiority, feature complexity or anything like this, do with the numbers what you like.
We ensured all of the arena resets correctly restored all blocks and block entities.
Also keep in mind that we only ran this with one bot. Outbound network traffic will be amplified by the amount of players, and likely so will allocation numbers, and to some extent, also CPU usage.

### A short conclusion
Besides AsyncArenasPlus and FastArena, almost all of these plugins act mainly as wrappers for FastAsyncWorldEdit. for this reason, their numbers are the same for basically all of these scenarios, just that some of these plugins produce higher overhead over FAWE with their reset logic. 
Also for this reason, besides WorldEdit, almost none of these plugins have a real impact on TPS, since they do as much of the Preparation for the reset on Async Threads. 
