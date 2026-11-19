# RISC-8 SoC: 8-bit 5-Stage CPU + Timer / SAR ADC

An 8-bit RISC CPU with a 5-stage pipeline (forwarding + hazard unit), extended with a memory-mapped
**8-bit Timer (PWM) + SAR ADC** peripheral. Written in Verilog, verified with directed tests, SVA and
UVM, and built in Vivado for the **XA7S100 (`xa7s100fgga676-1Q`)**.

```
                    risc8_soc
  +-----------------------------------------+
  |  risc8_5stage_core        timer_adc8_cpu_if
  |  IF-ID-EX-[MEM]-WB  --io_cs/we/addr/wdata-->  Timer | SAR ADC
  |  RAM 0x00-0xEF      <------io_rdata--------   FIFO(8) | 16 regs + IRQ
  +-----------------------------------------+
     clk shared, rst (active-high) -> ~rst -> peripheral rst_n
     outputs: pwm_out, irq, adc_sample / adc_ch / adc_dac, input: adc_cmp
```

## Features
- CPU talks to the peripheral through a 16-byte I/O page (`0xF0`-`0xFF`) with plain `LD` / `ST`.
- CPU change is about 15 lines: optional I/O port (`IO_ENABLE = 0` by default, so the original CPU is unchanged).
- Timer: prescaler, 8-bit counter, compare, one-shot, PWM output, overflow / compare events.
- SAR ADC: 4 channels, 8-bit, programmable sample window and clock divider, software / timer triggers.
- 8-entry result FIFO, window watchdog, 5 sticky IRQ flags (write-1-to-clear) and an `irq` pin.
- Demo firmware (`sw/soc_demo.s`) exercising timer, ADC, FIFO, flags and PWM.
- Synthesizable analog stand-in (`afe_digital_stub.v`), so the whole SoC runs on the FPGA without external parts.

## Register map (page `0xF0`-`0xFF`)
| Addr | Name | Access | Description |
|---|---|---|---|
| F0 | TMR_CTRL | RW | [0] EN, [1] ONESHOT |
| F1 | TMR_PRESC | RW | tick every PRESC+1 clocks |
| F2 | TMR_LOAD | RW | period = LOAD+1 ticks |
| F3 | TMR_CMP | RW | compare / PWM duty = CMP / (LOAD+1) |
| F4 | TMR_CNT | RO | current count |
| F5 | ADC_CTRL | RW | [0] EN, [1] START (self-clearing), [3:2] TRIG_SEL, [5:4] CH |
| F6 | ADC_SMP | RW | sample window = SMP+1 ticks |
| F7 | ADC_DIV | RW | ADC tick every DIV+1 clocks |
| F8 / F9 | ADC_THR_LO / HI | RW | window limits (reset 00 / FF) |
| FA | ADC_DATA | RO | oldest result, **read pops the FIFO** |
| FB | ADC_RCH | RO | [7] VALID, [1:0] CH of the oldest result (no pop) |
| FC | ADC_STAT | RO | [3:0] LEVEL, [4] EMPTY, [5] FULL, [6] BUSY |
| FD | IRQ_EN | RW | flag enables for the `irq` pin |
| FE | IRQ_STAT | W1C | [0] TMR_OVF, [1] TMR_CMP, [2] ADC_EOC, [3] ADC_WD, [4] FIFO_OVR |
| FF | ID | RO | always `0xA8` |

`TRIG_SEL`: 0 software only, 1 timer overflow, 2 timer compare, 3 either.
ADC code = `floor(Vin / 3.3 V * 256)`. All registers reset to 0 except `ADC_THR_HI` = `FF`.

## Repository layout
| Path | Content |
|---|---|
| `design_sources/` | CPU (`alu`, `regfile`, `imem`, `dmem`, `control_unit`, `hazard_unit`, `risc8_5stage_core`), `risc8_soc.v`, `risc8_fpga_top.v`, `afe_digital_stub.v` |
| `design_sources/periph/` | `timer8`, `sar_adc_ctrl`, `sync_fifo`, `apb_regs8`, `timer_adc8_top`, `timer_adc8_cpu_if` |
| `sim_sources/` | `tb_risc8_5stage` (CPU alone), `tb_timer_adc8` (peripheral), `tb_risc8_soc` (system), `tb_dump`, `afe_model` |
| `sv/` | `timer_adc_sva.sv` (34 assertions, attached with `bind`), `apb_if`, `adc_if`, `sar_afe_model` |
| `tb_uvm/` | UVM 1.2 testbench (`timer_adc_pkg.sv`, `tb_top.sv`) |
| `sw/` | `asm8.py` (assembler), `soc_demo.s` / `soc_demo.hex` (firmware), `iss.py` (golden model), `random_test.py` |
| `constraints/risc8_soc.xdc` | 100 MHz clock + configuration; pin assignments to be filled in |
| `scripts/` | `run_tb.tcl` (run one testbench), `run_uvm.tcl` (UVM runs), `create_project.tcl`, `run_synth_impl.tcl` |
| `docs/` | presentation slides (`.pptx`) |

## Getting started (Vivado 2026.1)
1. Create a project for part `xa7s100fgga676-1Q`.
2. Add `design_sources/**/*.v` (including `periph/`) as design sources and `constraints/risc8_soc.xdc` as constraints.
   Add `sw/soc_demo.hex` as a design source and set its type to **Memory Initialization Files**
   (the CPU ROM reads it with `$readmemh`). Set `risc8_fpga_top` as the top module.
3. Add `sim_sources/*.v` to the `sim_1` simulation set.
4. Pick a testbench: right-click it in **Simulation Sources > Set as Top**, and set
   `xsim.simulate.runtime` to `all`.

| Testbench | Needs the hex? | Expected final line |
|---|---|---|
| `tb_risc8_5stage` | no | `*** ALL TESTS PASSED ***` (51 cycles, CPI 1.50) |
| `tb_timer_adc8` | no | `*** PERIPHERAL TEST PASSED *** (39 checks)` |
| `tb_risc8_soc` | **yes** | `*** SOC TEST PASSED *** (22 checks)` |

`tb_risc8_soc` takes the firmware path from the `HEXFILE` generic, and the generic must be cleared before
running any other testbench. `scripts/run_tb.tcl` does this for you (it also closes the previous run,
sets the top, the runtime `all`, and launches the simulation). In the Vivado Tcl Console:
```tcl
source <path>/scripts/run_tb.tcl
run_tb tb_risc8_5stage
run_tb tb_timer_adc8
run_tb tb_risc8_soc <path>/sw/soc_demo.hex
```

### UVM testbench
`scripts/run_uvm.tcl` creates a separate `sim_uvm` simulation set (`sv/*.sv`, `tb_uvm/*.sv`, top `tb_top`),
adds the XSim options `-L uvm` (Vivado ships UVM) and launches the chosen test. With the project open:
```tcl
source <path>/scripts/run_uvm.tcl
list_uvm_tests                ;# show the available tests
run_uvm                       ;# tadc_all_test, seed 1
run_uvm tadc_timer_test       ;# one test
run_uvm tadc_random_test 7    ;# one test with a seed
close_uvm                     ;# close the simulation
```
A passing run prints `*** UVM TEST PASSED ***`, `UVM_ERROR : 0` and `UVM_FATAL : 0`.

Structure: `test -> env -> apb_agent (sequencer, driver, monitor) + adc_agent (voltage setter, conversion monitor)
+ scoreboard (register and FIFO reference model) + 2 coverage collectors`, driving `timer_adc8_top`
with the behavioural `sar_afe_model`. The 34 assertions in `sv/timer_adc_sva.sv` are bound into the DUT.

| Test | What it covers |
|---|---|
| `tadc_reg_test` | reset values, register read/write, ID |
| `tadc_adc_sw_test` | software-started conversions: corner and random voltages, 4 channels, several dividers |
| `tadc_timer_test` | period, PWM duty, one-shot, OVF / CMP flags |
| `tadc_adc_timer_test` | timer-triggered conversions (overflow, compare, either) |
| `tadc_window_test` | window limits, ADC_WD flag, `irq` pin |
| `tadc_fifo_test` | fill to 8 entries, FIFO_OVR flag, drain |
| `tadc_random_test` | random mix of conversions and register traffic |
| `tadc_all_test` | all of the above in sequence |

### Firmware
```
python sw/asm8.py sw/soc_demo.s sw/soc_demo.hex
```
Registers are assembled to 16-bit words, one per line, for `$readmemh`.

## Results
| Item | Result |
|---|---|
| CPU alone (XSim) | PASS, 51 cycles, CPI 1.50 |
| Peripheral testbench (XSim) | PASS, 39 checks |
| System test (XSim) | PASS, 22 checks, 639 cycles, 122 I/O accesses |
| UVM (XSim) | all tests PASS |
| Utilization (post-synthesis) | 482 LUT (0.75 %), 413 FF (0.32 %), 32 LUTs as distributed RAM, 0 BRAM, 0 DSP, 0 latches |
| Timing, 100 MHz constraint | WNS +1.691 ns, WHS +0.057 ns, WPWS +3.750 ns, 0 failing of 1130 endpoints |

Also checked with open-source tools (Icarus Verilog 12, Verilator 5.020): 500 random programs match
`iss.py` (CPU with `IO_ENABLE = 0`, `sim_sources/tb_dump.v`), Verilator `-Wall` lint is clean, and the
34 assertions show no failures. Planted integration mistakes (reset not inverted, missing load mux,
`io_cs` not gated with LD/ST, wrong I/O page) are all caught by the system test.

## Limitations
- The CPU has no interrupt input (all 16 opcodes are used), so firmware polls; `irq` is a pin only.
- Pin assignments are not provided, so no bitstream is generated yet. The analog front end is a digital stub.
- `scripts/create_project.tcl` and `scripts/run_synth_impl.tcl` have not been run on a real Vivado install (the project was built by adding the files by hand).

## Possible next steps
Real interrupts (irq input, EPC register, `RETI`), reset synchronizer, pin constraints and a board demo
with a real DAC and comparator, UVM register model (RAL).
