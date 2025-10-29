# Common Implementation Notes for HA Modbus Add-ons

This file contains shared implementation notes and requirements for all add-ons in this repository (ha-paneltrack, ha-sungrow, ha-atess, ha-kehua, etc.).

## Required: ModbusException Handling in Client.read()

**Status**: Implemented in ha-paneltrack as of 2025-10-29

All add-ons that use pymodbus MUST handle both types of errors when reading registers:

### 1. ModbusException (Connection/Communication Failures)
- **When it occurs**: Connection timeouts, network errors, device offline, communication failures
- **Type**: Python exception raised by pymodbus library
- **Handling**: Must be caught with try/except

### 2. result.isError() (Device-Reported Errors)
- **When it occurs**: Device successfully received request but rejected it (illegal address, illegal function, etc.)
- **Type**: Valid Modbus response with error flag
- **Handling**: Must check after successful read

### Implementation Pattern

In `client.py`, the `read()` method should wrap pymodbus calls:

```python
from pymodbus import ModbusException

def read(self, address, count, slave_id, register_type):
    try:
        if register_type == RegisterTypes.HOLDING_REGISTER:
            result = self.client.read_holding_registers(...)
        elif register_type == RegisterTypes.INPUT_REGISTER:
            result = self.client.read_input_registers(...)
        return result
    except ModbusException as exc:
        logger.error(f"ModbusException reading slave {slave_id} at address {address}: {exc}")
        raise  # Re-raise to allow server to handle gracefully
```

Then in `server.py`, `read_registers()` should handle both error types:

```python
try:
    result = self.connected_client.read(address, count, slave_id, register_type)
except ModbusException as exc:
    # Connection failure - device likely offline
    raise Exception(f"Connection error reading {parameter_name}")

if result.isError():
    # Device-reported error
    self.connected_client._handle_error_response(result)
    raise Exception(f"Device error reading {parameter_name}")
```

### Why This Matters

Without proper ModbusException handling:
- When a single device goes offline, the entire app crashes
- All other devices stop being monitored
- No graceful degradation or reconnection attempts

With proper handling:
- Failed devices can be moved to a reconnection queue
- Other devices continue operating normally
- Offline availability status can be published to MQTT

## Action Required

Other add-ons (ha-sungrow, ha-atess, ha-kehua) should implement this pattern to ensure robust operation when devices go offline.
