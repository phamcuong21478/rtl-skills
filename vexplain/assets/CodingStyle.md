
# Verilog Coding Style

The following rules must be followed.

## General File Appearance

- Wrap the code at 100 characters per line. Exceptions: Any place where line wraps are impossible (for example, an include path might extend past 100 characters).

- Do not use tabs anywhere. Use four spaces for indentation.

- Remove trailing whitespace at the end of each line.

- Use begin and end unless the whole statement fits on a single line.

## Signal Naming

Signal names must follow the format: `[<pipeline_name>_][<pipeline stage>_]<signal_name>[_ff]`

where
- `<pipeline_name>` is the name of pipeline process. Use this field only if the design contains more than one pipeline process. It helps distinguish between different pipeline paths. This field must be concise, no more than 3 characters. 
- `<pipeline_stage>` can be `s0`, `s1`, `s2`... corresponding to stages 0, 1, 2... Stage `s0` represents the cycle in which the pipeline input becomes valid.
- `[_ff]` is only used when the signal is expected to synthesize into a flip-flop or register.

Example:
- `s0_cnt` : the `cnt` signal at stage 0 (corresponding to the valid input)
- `req_ff` : the `req` signal not according to the pipeline design, which will be synthesized into a flip-flop

If the design has pipeline process, you must add `[<pipeline stage>_]` to all names, except for memory signals and in/out ports.

## Bus Naming

- Bus names should be kept as short as possible. Examples: `axi`, `upi`, `ob00`.
- For input/output ports, use the following prefixes:
  - `s_` for slave interfaces
  - `m_` for master interfacesfor example, an include path might extend past 100 characters.

- Do not use tabs anywhere. Use four spaces for indentation.

- Remove trailing whitespace at the end of each line.

  - `b_` for monitor interfaces
- If a bus is used as an internal connection between submodules, its name must follow this format `<master>2<slave>_<bus_name>[_<bus_inst>]`. Where:
  - `<master>` is the abbreviated instance name of the master module and must not exceed 3 characters
  - `<slave>` is the abbreviated instance name of the slave module and must not exceed 3 characters
  - `<bus_name>` is the short name of the bus type
  - `[_<bus_inst>]` is optionasádl. This field should be used only when more than one bus of the same type connects the same pair of modules, in order to distinguish between them.

## Others

- Moore FSM: single `always` block for both state and output
- Mealy FSM: one `always` block for state, separate `always @(*)` block for output logic.
- Constant/FSM states must use localparam with explicit values:
```
localparam FSM_IDLE   = 'd0,
           FSM_ENCODE = 'd1,
           FSM_DONE   = 'd2;
```

## Comment

- C++ style comments (`// foo`) are preferred. C style comments (`/* bar */`) can also be used.

- split code section by commend like bellow
```
//==============================================================================
// <code section>
//==============================================================================
```

