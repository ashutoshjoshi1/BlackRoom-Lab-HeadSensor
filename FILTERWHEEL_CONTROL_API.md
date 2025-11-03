# Filter Wheel Control API - Technical Documentation

## Overview

This document provides complete technical specifications for controlling Pandora spectrometer filter wheels via RS-232 serial communication. This API is designed for integration into external software systems.

## Architecture

### Communication Protocol
- **Interface**: RS-232 Serial Communication
- **Baud Rate**: Configurable (typically 4800 baud)
- **Data Bits**: 8
- **Parity**: None (N)
- **Stop Bits**: 1
- **Flow Control**: None
- **Timeout**: 0 (non-blocking, check answers via polling loop)
- **Write Timeout**: 20 seconds

### Hardware Configuration
- **Filter Wheel 1 (FW1)**: Positions 1-9
- **Filter Wheel 2 (FW2)**: Positions 1-9 (optional, may not be present)
- **Device ID**: Configurable identifier (e.g., "Pan70HST")

## Core Function: set_headsensor()

### Function Signature

```python
def set_headsensor(hst_pars, part, pos="r", callfree=0, callinfo=[], stupdate=True):
    """
    Sets filterwheel 1 (FW1), filterwheel 2 (FW2), or shadowband (SB) position.
    
    Parameters:
    -----------
    hst_pars : list
        Head sensor-tracker parameters list. Required structure:
        - hst_pars[0]: Serial port handle (pySerial Serial object or integer <=-9 for simulation)
        - hst_pars[1]: Connection type string ("RS232")
        - hst_pars[3]: Device ID string (e.g., "Pan70HST") - expected response for ID queries
        - hst_pars[14]: Filter wheel 1 offset (integer, typically 0)
        - hst_pars[21]: Auto-search mode flag (0=auto, 1=manual port)
        
    part : str
        Component identifier:
        - "F1" for Filter Wheel 1
        - "F2" for Filter Wheel 2
        - "SB" for Shadowband (optional, not filter wheel)
        
    pos : int or str
        Position command:
        - Integer 1-9: Move to that position
        - "r": Reset filter wheel (returns to home position)
        - For shadowband: Integer -1000 to 1000 (step position)
        
    callfree : int, optional
        Port release behavior:
        - 0: Free port after completion (default)
        - 1: Keep port busy (for chained operations)
        - 2: Chain to callback function
        
    callinfo : list, optional
        Callback information for callfree=2:
        - callinfo[0]: Function name string to call
        - callinfo[1]: Argument list for callback
        
    stupdate : bool, optional
        Update status messages (default: True)
        
    Returns:
    --------
    None (status is checked via serial_status["hst"])
    """
```

### Command Format

The function generates ASCII commands sent over RS-232:

**Filter Wheel Position Command:**
```
F1{position}\r
F2{position}\r
```
- Examples: `F15\r` (move FW1 to position 5), `F23\r` (move FW2 to position 3)

**Filter Wheel Reset Command:**
```
F1r\r
F2r\r
```

### Expected Response Format

The sensorhead responds with a structured answer:

**Success Response:**
```python
polished_answer = ["F1", [0]]  # or ["F2", [0]]
```
- Element 0: Device ID ("F1" or "F2")
- Element 1: List with error code
  - `[0]`: Success
  - `[1]`, `[2]`, etc.: Error codes (see Error Handling section)

**Error Response:**
```python
polished_answer = ["F1", [error_code]]
```

**Device ID Check Response:**
- String match against `hst_pars[3]` (e.g., "Pan70HST")

## Status Management

### Serial Status Structure

Filter wheel operations use the `serial_status["hst"]` dictionary/list structure:

```python
serial_status["hst"] = [
    0,      # [0] Low-level status: -2=lost, -1=error, 0=free, 1/4=free-low-level, 2=busy, 3=answer-received, 9=waiting
    0,      # [1] High-level status: 0=none, 1=initiate, 2=in-progress, 3=waiting
    0,      # [2] Time of last command (high-level)
    0,      # [3] Maximum timeout for operation [seconds]
    [],     # [4] Action details: [ID, value] where ID="F1"/"F2", value=position or "r"
    "",     # [5] Last command sent (low-level)
    [],     # [6] Expected answer format
    0,      # [7] Timeout for low-level communication [seconds]
    [],     # [8] Communication history (list of transactions)
    [0, 5], # [9] Unexpected answers: [count, max_allowed]
    [       # [10] Recovery information
        0,      # [0] Current recovery level (0=none, 1-4=recovery)
        -1,     # [1] Recovery limit (-1=auto, >=0=max level, -2=abort, -3=forced abort)
        -1,     # [2] Last logged recovery level
        "",     # [3] Current recovery action description
        [],     # [4] Recovery attempt counters per level
        [],     # [5] Recovery attempt counters for set_tracker
        []      # [6] Recovery attempt counters for get_tracker
    ],
    "",     # [11] Callback function name
    [],     # [12] Callback function arguments
    "",     # [13] Error log string
    "OK"    # [14] Final error message ("OK" or error description)
]
```

### Status Checking

**Check if operation is complete:**
```python
if serial_status["hst"][0] == 0 and serial_status["hst"][-1] == "OK":
    # Operation successful and port is free
    pass
elif serial_status["hst"][0] == -1:
    # Error occurred
    error_message = serial_status["hst"][-1]
```

**Check current filter wheel position:**
```python
current_action = serial_status["hst"][4]
if len(current_action) == 2:
    fw_id = current_action[0]      # "F1" or "F2"
    fw_position = current_action[1] # Position number or "r"
```

## Low-Level Serial Communication

### Query-Answer Function (Internal)

The `qa()` function handles low-level serial I/O:

```python
def qa(port_handle, uselog="HS"):
    """
    Sends command from serial_status["hst"][5] and receives answer.
    Updates serial_status["hst"][8] with communication history.
    
    Parameters:
    -----------
    port_handle : Serial object or int
        Serial port handle (from hst_pars[0])
        If <= -9: Simulation mode
        
    uselog : str
        Logging identifier ("HS" for head sensor)
    """
```

**Process:**
1. Read any buffered data from serial port
2. Send command string (from `serial_status["hst"][5]`) + `\r`
3. Poll for response until timeout or answer received
4. Parse raw response into polished answer format
5. Store transaction in `serial_status["hst"][8]`
6. Update status flags

### Raw Response Parsing

**Example raw responses:**
- Success: `"F10\r\n"` → `["F1", [0]]`
- Error: `"F12\r\n"` → `["F1", [2]]`
- Device ID: `"Pan70HST\r\n"` → `"Pan70HST"`

**Parsing logic:**
1. Strip `\r\n` terminators
2. Check if response matches expected format (list or string)
3. Extract device ID and error code
4. Validate against expected answer (from `serial_status["hst"][6]`)

## Error Handling and Recovery

### Error Codes

Common error codes returned by sensorhead:
- **0**: Success/OK
- **1**: Communication error
- **2**: Hardware error / blocked
- **99**: Low-level serial communication error (parsing failed)

### Recovery Levels

The system implements 4-level recovery for filter wheel operations:

**Level 0: Normal Operation**
- Send position command directly
- Command: `F1{pos}` or `F1r` for reset

**Level 1: Reset Recovery**
- Perform filter wheel reset before positioning
- Commands: `F1r` (reset), then retry position command

**Level 2: Communication Check**
- Verify serial communication is working
- Command: Device ID query (expects `hst_pars[3]` response)

**Level 3: Port Reset**
- Close and reopen serial port
- Re-establish connection

**Level 4: Wait and Retry**
- Wait timeout period, then recall function
- Maximum 5 repetitions before abort

### Recovery Behavior

**Automatic Recovery:**
- On error, recovery level increases
- On success after recovery, level decreases
- Maximum recovery attempts tracked in `serial_status["hst"][10][4]`

**Manual Abort:**
```python
serial_status["hst"][10][1] = -2  # User abort
serial_status["hst"][10][1] = -3  # Forced abort (max attempts reached)
```

## Complete Integration Example

### Python Implementation

```python
import serial
import time
from threading import Timer

class FilterWheelController:
    def __init__(self, port_name, baudrate=4800, device_id="Pan70HST"):
        """
        Initialize filter wheel controller.
        
        Parameters:
        -----------
        port_name : str
            Serial port name (e.g., "COM3" on Windows, "/dev/ttyUSB0" on Linux)
        baudrate : int
            Serial baud rate (default: 4800)
        device_id : str
            Expected device ID string for communication verification
        """
        # Initialize serial port
        self.serial_port = serial.Serial(
            port=port_name,
            baudrate=baudrate,
            bytesize=8,
            parity='N',
            stopbits=1,
            timeout=0,  # Non-blocking
            write_timeout=20
        )
        
        # Initialize hst_pars structure
        self.hst_pars = [
            self.serial_port,    # [0] Port handle
            "RS232",             # [1] Connection type
            None,                # [2] (unused)
            device_id,           # [3] Device ID
            None,                # [4] (unused)
            [],                  # [5] FW1 filter names (from config)
            [],                  # [6] FW2 filter names (from config)
            None,                # [7] Tracker type
            0.01,                # [8] Tracker resolution
            None,                # [9] Tracker limits
            None,                # [10] (unused)
            None,                # [11] (unused)
            0.004,               # [12] Shadowband resolution
            0.0,                 # [13] Shadowband offset
            0,                   # [14] FW1 offset
            None,                # [15-20] (other parameters)
            0                    # [21] Auto-search mode (0=auto, 1=manual)
        ]
        
        # Initialize serial status
        self.serial_status = {
            "hst": [
                0,      # [0] Low-level status
                0,      # [1] High-level status
                0,      # [2] Time of last command
                5.0,    # [3] Maximum timeout (5 seconds)
                [],     # [4] Action details
                "",     # [5] Last command
                [],     # [6] Expected answer
                3.0,    # [7] Low-level timeout
                [],     # [8] Communication history
                [0, 5], # [9] Unexpected answers
                [       # [10] Recovery info
                    0, -1, -1, "", [0, 0, 0, 0, 0], [], []
                ],
                "",     # [11] Callback
                [],     # [12] Callback args
                "",     # [13] Error log
                "OK"    # [14] Final error message
            ]
        }
        
        # Error code mapping
        self.error_codes = [
            ["OK", "No error"],
            ["Error 1", "Communication error"],
            ["Error 2", "Hardware error / blocked"],
            ["Error 99", "Low-level serial communication error"]
        ]
    
    def set_filterwheel(self, fw_number, position, reset=False, timeout=5.0):
        """
        Set filter wheel position.
        
        Parameters:
        -----------
        fw_number : int
            Filter wheel number (1 or 2)
        position : int or str
            Position (1-9) or "r" for reset
        reset : bool
            If True, perform reset before positioning
        timeout : float
            Maximum time to wait for completion [seconds]
        
        Returns:
        --------
        bool : True if successful, False if error
        str : Error message ("OK" if successful)
        """
        part = f"F{fw_number}"
        
        # Handle reset
        if reset or position == "r":
            pos = "r"
        elif isinstance(position, int) and 1 <= position <= 9:
            pos = position
        else:
            return False, f"Invalid position: {position}"
        
        # Reset status
        self.serial_status["hst"][0] = 1  # Set to "free for low level only"
        self.serial_status["hst"][-1] = "OK"
        self.serial_status["hst"][3] = timeout
        self.serial_status["hst"][4] = [part, pos]
        
        # Prepare command
        if pos == "r":
            command = f"{part}r"
        else:
            command = f"{part}{pos}"
        
        self.serial_status["hst"][5] = command
        self.serial_status["hst"][6] = [part, [[0], [1], [2], [99]]]  # Expected answers
        self.serial_status["hst"][7] = 3.0  # Low-level timeout
        
        # Send command
        try:
            # Clear input buffer
            self.serial_port.reset_input_buffer()
            
            # Send command
            cmd_bytes = (command + "\r").encode('ascii')
            self.serial_port.write(cmd_bytes)
            
            # Wait for response
            start_time = time.time()
            response = b""
            while (time.time() - start_time) < self.serial_status["hst"][7]:
                if self.serial_port.in_waiting > 0:
                    response += self.serial_port.read(self.serial_port.in_waiting)
                    if b"\r\n" in response:
                        break
                time.sleep(0.01)  # Small delay to avoid CPU spinning
            
            # Parse response
            if response:
                response_str = response.decode('ascii', errors='ignore').strip()
                # Parse response format: "F1{error_code}\r\n"
                if len(response_str) >= 3 and response_str[:2] == part:
                    try:
                        error_code = int(response_str[2:])
                        self.serial_status["hst"][0] = 3  # Answer received
                        
                        if error_code == 0:
                            self.serial_status["hst"][-1] = "OK"
                            self.serial_status["hst"][0] = 0  # Free
                            return True, "OK"
                        else:
                            error_msg = self._get_error_message(error_code)
                            self.serial_status["hst"][-1] = error_msg
                            self.serial_status["hst"][0] = -1  # Error
                            return False, error_msg
                    except ValueError:
                        self.serial_status["hst"][-1] = "Parse error"
                        self.serial_status["hst"][0] = -1
                        return False, "Failed to parse response"
                else:
                    self.serial_status["hst"][-1] = "Unexpected response format"
                    self.serial_status["hst"][0] = -1
                    return False, "Unexpected response format"
            else:
                self.serial_status["hst"][-1] = "Timeout"
                self.serial_status["hst"][0] = -1
                return False, "No response received"
                
        except Exception as e:
            self.serial_status["hst"][-1] = f"Exception: {str(e)}"
            self.serial_status["hst"][0] = -1
            return False, f"Exception: {str(e)}"
    
    def _get_error_message(self, error_code):
        """Map error code to message."""
        if error_code == 0:
            return "OK"
        elif error_code < len(self.error_codes):
            return self.error_codes[error_code][1]
        else:
            return f"Unknown error code: {error_code}"
    
    def reset_filterwheel(self, fw_number, timeout=5.0):
        """Reset filter wheel to home position."""
        return self.set_filterwheel(fw_number, "r", timeout=timeout)
    
    def close(self):
        """Close serial port."""
        if self.serial_port and self.serial_port.is_open:
            self.serial_port.close()

# Usage example
if __name__ == "__main__":
    # Initialize controller
    controller = FilterWheelController("COM3", baudrate=4800, device_id="Pan70HST")
    
    try:
        # Move filter wheel 1 to position 5
        success, message = controller.set_filterwheel(1, 5)
        if success:
            print(f"Filter wheel 1 moved to position 5: {message}")
        else:
            print(f"Error: {message}")
        
        # Reset filter wheel 2
        success, message = controller.reset_filterwheel(2)
        if success:
            print(f"Filter wheel 2 reset: {message}")
        else:
            print(f"Error: {message}")
    
    finally:
        controller.close()
```

## Configuration File Format

Filter wheel positions are defined in operation files with this format:

```
Filterwheel 1, position 1 -> OPEN
Filterwheel 1, position 2 -> OPEN
Filterwheel 1, position 3 -> ND3
Filterwheel 1, position 4 -> OPEN
Filterwheel 1, position 5 -> ND1
Filterwheel 1, position 6 -> ND4
Filterwheel 1, position 7 -> ND2
Filterwheel 1, position 8 -> ND2
Filterwheel 1, position 9 -> ND5
Filterwheel 2, position 1 -> OPEN
Filterwheel 2, position 2 -> DIFF
Filterwheel 2, position 3 -> OPAQUE
...
```

**Valid filter names:**
- `OPEN` - No filter (open position)
- `ND1` to `ND5` - Neutral density filters
- `DIFF` - Diffuser
- `OPAQUE` - Opaque (blocking)
- `U340` - UV filter
- `BP300` - Bandpass filter
- Custom names allowed

## Timing Considerations

### Operation Durations
- **Filter wheel position change**: ~1.0 second
- **Filter wheel reset**: ~5.0 seconds
- **Serial command timeout**: 3.0 seconds (low-level)
- **Maximum operation timeout**: 5.0 seconds (high-level, configurable)

### Best Practices
1. **Wait for completion**: Check `serial_status["hst"][0] == 0` before next command
2. **Handle errors**: Always check `serial_status["hst"][-1]` for error messages
3. **Poll status**: For non-blocking operation, poll status periodically
4. **Timeout handling**: Implement timeout in your application layer
5. **Error recovery**: Implement retry logic with exponential backoff

## Advanced Features

### Chained Operations

For chaining multiple filter wheel operations:

```python
# First operation: keep port busy
controller.set_filterwheel(1, 5, callfree=1)

# Wait for completion
while controller.serial_status["hst"][0] != 0:
    time.sleep(0.1)

# Second operation: immediate
controller.set_filterwheel(2, 3, callfree=0)
```

### Status Polling

Non-blocking status check:

```python
def wait_for_completion(controller, timeout=5.0):
    """Wait for filter wheel operation to complete."""
    start_time = time.time()
    while True:
        status = controller.serial_status["hst"][0]
        error = controller.serial_status["hst"][-1]
        
        if status == 0:  # Free (complete)
            return error == "OK"
        elif status == -1:  # Error
            return False
        
        if (time.time() - start_time) > timeout:
            return False
        
        time.sleep(0.1)
```

## Troubleshooting

### Common Issues

**1. No Response**
- Check serial port connection
- Verify baud rate matches sensorhead configuration
- Check cable connections
- Verify device is powered

**2. Timeout Errors**
- Increase timeout values
- Check for stuck filter wheel (may need reset)
- Verify serial port is not being used by another process

**3. Parse Errors**
- Verify response format matches expected structure
- Check for corrupted serial data (cable issues)
- Implement retry logic

**4. Device Not Found**
- Verify device ID matches configuration
- Check auto-search vs manual port configuration
- Verify serial port number is correct

## API Summary

### Essential Functions

1. **Initialization**
   - Configure serial port
   - Initialize `hst_pars` structure
   - Initialize `serial_status["hst"]`

2. **Control**
   - `set_filterwheel(fw_number, position, reset=False)`
   - `reset_filterwheel(fw_number)`

3. **Status**
   - Check `serial_status["hst"][0]` for completion
   - Check `serial_status["hst"][-1]` for errors

4. **Cleanup**
   - Close serial port when done

### Command Reference

| Action | Command String | Expected Response |
|--------|---------------|-------------------|
| Move FW1 to position N | `F1N\r` | `F10\r\n` (success) or `F1E\r\n` (error E) |
| Move FW2 to position N | `F2N\r` | `F20\r\n` (success) or `F2E\r\n` (error E) |
| Reset FW1 | `F1r\r` | `F10\r\n` |
| Reset FW2 | `F2r\r` | `F20\r\n` |

Where N = 1-9, E = error code (1, 2, etc.), and `\r\n` are line terminators.

## Notes

- All commands end with `\r` (carriage return)
- All responses end with `\r\n` (carriage return + line feed)
- Commands are case-sensitive
- Position numbers must be single digits (1-9)
- Reset operations take longer than position changes
- Serial port must be opened before sending commands
- Non-blocking I/O requires polling for responses

