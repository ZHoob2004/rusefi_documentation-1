# Lua Scripting

# Basics

A proper ECU has to offer users as much flexibility as possible, meaning a completely user-defined control strategy for both primary and auxiliary actuators. For many years rusEFI had FSIO to do just that.

As of 2021, rusEFI is replacing FSIO with a popular open source [Lua scripting engine](https://en.wikipedia.org/wiki/Lua_(programming_language))
We use [Lua](https://www.lua.org/) version 5.4.3

## Conventions

- The Lua interpreter will trigger an error if there is a mistake in the program, check the rusEFI console to see errors and script output.
- Unless otherwise mentioned, all `index` parameters start with the first element at index at 0.

## Writing Your Script

The entire Lua script is read at startup, then a function called `onTick` is called periodically by rusEFI.

Here is a simple script you can run to illustrate this behavior:

```
print('Hello Lua startup!')

function onTick()
    print('Hello onTick()')
end
```

### Controlling the Tick Rate

The function `setTickRate(hz)` can be used to configure how often rusEFI calls the `onTick` function.  If your script does a lot of work in the `onTick()` function it may run slower than the desired rate.  Since the Lua virtual machine runs at low priority compared to other functions of the ECU, it is impossible to swamp the ECU with too much Lua work, so set the tick rate to whatever is necessary. `onCanRx` runs at the same rate as `onTick`

```
n = 0
setTickRate(5) --set tick rate to 5hz
function onTick()
    print('Hello Lua: ' ..n)
    n = n + 1
end
```

# Function Reference

## User Settings

### `getOutput(name)`

For example ``getOutput("clutchUpState")`` ``getOutput("brakePedalState")``

See [https://github.com/rusefi/rusefi/blob/master/firmware/controllers/lua/generated/output_lookup_generated.cpp](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/lua/generated/output_lookup_generated.cpp) for output names.

### `setClutchUpState(value)`

### `setBrakePedalState(value)`

### `getCalibration(name)`

Gets current calibration value for specified scalar setting. For example ``getCalibration("cranking.rpm")``

See [https://github.com/rusefi/rusefi/blob/master/firmware/controllers/lua/generated/value_lookup_generated.cpp](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/lua/generated/value_lookup_generated.cpp) for field names.

### `setCalibration(name, value, needEvent)`

Sets specified calibration setting to specified value. Fires calibration change event depending on needEvent parameter.

For example ``setCalibration("cranking.rpm", 900, false)``

### `findSetting(name, defaultValue)`
Find User Setting with specified name and returns numeric value. Useful when script developer and script consumer are
different people, also useful while Lua script editing is available only in TS.

- Parameters
 - `name`: Variable name, as in corresponding 'name' field in configuration
 - `dataultValue`: value to use if specified setting not located by name

## Engine Control

### `stopEngine`

### `setSparkSkipRatio`

setSparkSkipRatio(0) to skip 0% of the ignition events, i.e. no skipping
setSparkSkipRatio(0.5) would skip half of ignition events. We never skip two consecutive ignitions.

### `setTimingAdd(angle)`
todo add details but ready to test!

### `setTimingMult(coeff)`

todo add details but ready to test!

### `setEtbAdd(extraEtb)`

extraEtb `10` for 10%

## CAN bus

### `enableCanTx(isEnabled)`

enabled by default

use enableCanTx(false) to suppress CAN TX

### `txCan(bus, ID, isExt, payload)`
- Parameters
 - bus: hardware CAN bus index, only '1' on most rusEFI boards, '1' or '2' on Proteus
 - isExt: 0 for 11 bit mode


## Utility

### `print(msg)`

Print a line of text to the ECU's log.

- Parameters
  - `msg`: The message to print.  Pass a string or number and it will be printed to the log.
- Returns
  - none

#### Usage example

Program:
```
n = 5.5
print('Hello Lua, number is: ' ..n)
```
Output:
`Hello Lua, number is 5.5`

### `setTickRate(hz)`

Sets the rate at which rusEFI calls your `onTick` and `onCanRx` functions, in hz. On reset default is 10hz.

- Parameters
  - `hz`: Desired tick rate, in hz.  Values passed will be clamped to a minimum of 1hz, and maximum of 100hz.
- Returns
  - none

### `mcu_stop`

Stops MCU.

### `table3d(tableIdx, x, y)`

Looks up a value from the specified Script Table.

- Parameters
  - `tableIdx`: Index of the table to use.  Currently 4 tables are supported, so indices 1, 2, 3, and 4 are valid.
  - `x`: X-axis value to look up in the table (this is often RPM)
  - `y`: Y-axis value to look up in the table (this is often load)
- Returns
  - A number representing the value looked up from the table.

### `curve(curveIdx, x)`

Looks up a value from the specified Script Curve.
- Parameters
  - `tableIdx`: Index of the script to use, starting from 1.
  - `x`: Axis value to look up in the table


### `setDebug(index, value)`

Sets a debug channel to the specified value.  Note: this only works when the ECU debug mode is set to `Lua`.

- Parameters
  - `index`: the index of the debug channel to set, 1 thru 7 inclusive.
  - `value`: the value to set the channel to
- Returns
  - none

### `mcu_standby`

Puts MCU into standby low current consumption mode.

## Input

### `getSensor(name)`

Reads the specified sensor.

- Parameters
  - `name`: Name of the sensor to read.  [A list of sensor indices can be found here.](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/sensors/sensor_type.h)
- Returns
  - A reading from the sensor, or `nil` if the sensor has a problem or isn't configured.

### `getSensorByIndex(index)`

Reads the specified sensor.

- Parameters
  - `index`: Index of the sensor to read.  [A list of sensor indices can be found here.](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/sensors/sensor_type.h)
- Returns
  - A reading from the sensor, or `nil` if the sensor has a problem or isn't configured.

### `getSensorRaw(index)`

Reads the raw value from the specified sensor.  For most sensors, this means the analog voltage on the relevant input pin.

- Parameters
  - `index`: Index of the sensor to read.  [A list of sensor indices can be found here.](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/sensors/sensor_type.h)
- Returns
  - The raw value that yielded the sensor reading, or 0 if the sensor doesn't support raw readings, isn't configured, or has failed.

### `getAuxAnalog(index)`

More or less like getSensorRaw but always voltage of aux analog input. 

- Parameters
  - `index`: Index of aux analog sensor to read. From 0 to 3
- Returns
  - Voltage of sensor reading, or nil if sensor isn't configured.

### `hasSensor(index)`

Checks whether a particular sensor is configured (whether it is currently valid or not).

- Parameters
  - `index`: Index of the sensor to check.  [A list of sensor indices can be found here.](https://github.com/rusefi/rusefi/blob/master/firmware/controllers/sensors/sensor_type.h)
- Returns
  - A boolean value, `true` if the sensor is configured, and `false` if not.

### `getDigital(index)`

Reads a digital input from the specified channel.

- Parameters
  - `index`: The index of the digital channel to read.  See table below for values.
- Returns
  - A boolean value representing the state of the input pin.  `true` = high voltage (above ~2 volts), `false` = low voltage (below ~3 volts)

Valid `index` parameter values:

| Index | Channel Name |
| --- | ---:|
| 0 | Clutch down switch |
| 1 | Clutch up switch |
| 2 | Brake switch |
| 3 | AC switch |

### `readPin(pinName)`

Reads physical value of arbitrary MCU pin
- Parameters
 - `pinName`: string name of MCU pin, for examples "PD15"

## Output

### `startPwm(index, frequency, duty)`

Initializes PWM on the specified index, starting at the specified frequency and duty cycle.  The index selects which config field pin to use, see "Lua PWM Outputs" page in TunerStudio.

- Parameters
  - `index`: The index of the PWM channel to start.  Valid values are 0 through 7, one for each of the 8 channels.
  - `frequency`: Initial frequency of the output, in hertz (cycles per second).  Valid values are between 1 and 1000hz.
  - `duty`: Initial duty cycle of the output.  `0.0` = fully off, and `1.0` = fully on.  `0.25` = on 25% of the time, off 75% of the time.
- Returns
  - none

### `setPwmDuty(index, duty)`

Set the duty cycle of the specified PWM channel.

- Parameters
  - `index`: The index of the PWM channel to set.
  - `duty`: Desired duty cycle of the output.  `0.0` = fully off, and `1.0` = fully on.  `0.25` = on 25% of the time, off 75% of the time.
- Returns
  - none

### `setPwmFreq(index, frequency)`

- Parameters
  - `index`: The index of the PWM channel to set.
  - `frequency`: Initial frequency of the output, in hertz (cycles per second).  Valid values are between 1 and 1000hz.
- Returns
  - none


## Misc console commands

``luamemory``

``luareset``


## Examples

# timer
```
t = Timer.new();
timingAdd = 0;

function onTick()
   auxV = getAuxAnalog(0)
   tps = getSensor("TPS")
-- todo: check for NIL value which is a sign of analog input not assigned in TS
   if auxV > 2 then
     t:reset();
   end

   val = t:getElapsedSeconds();

   if t:getElapsedSeconds() < 3 then
     timingAdd = 10;
   else
     timingAdd = 0;
   end
   setTimingAdd(timingAdd)


   print('Hello analog ' .. auxV .. " " .. val)
end
```



# PWM
```
-- index 0, 100Hz, zero duty inititally
startPwm(0, 100, 0)

function onTick()
 enable_pump = getSensor("RPM") > 700  and getSensor("BatteryVoltage") > 13 and getSensor("VehicleSpeed") <60
 
 -- lua does not have ternary ? : operator, this here means "1 if enable_pump and 0 otherwise"
 setPwmDuty(0, enable_pump and 1 or 0)

end
```

# CAN transmit
```
function onTick()
  tps = getSensor("CLT")
  print('TPS ' .. tps)
  voltage0 = getSensor("aux0")
 
  txPayload = {}
  // first byte: integer part, would be autoboxed to int
  txPayload[1] = voltage0
  // second byte: fractional part, would be autoboxed to int, overflow would be ignored
  txPayload[2] = voltage0 * 256;

  txCan(1, 0x600, 1, txPayload)

end
```

# CAN receive
```
canRxAdd(0x500)
canRxAdd(0x570)
function onCanRx(bus, id, dlc, data)
	print('got CAN id=' ..id ..' dlc=' ..dlc)
	id11 = id % 2048
	if id11 == 0x500 then
		-- Check can state of BCM
		canState = data[1]
	end
	if id11 == 0x570 then
		mcu_standby()
	end
end
```


```
function decimalToHex(num)
	if num == 0 then
		return '0'
	end

	local hexstr = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, "A", "B", "C", "D", "E", "F" }
	local result = ""
	while num > 0 do
		local n = num % 16
		result = hexstr[n + 1] ..result
		num = math.floor(num / 16)
	end
	return result
end



function print_array(arr)
	local str = ""
	local index = 1
	while arr[index] ~= nil do
		str = str.." "..decimalToHex(arr[index])
		index = index + 1
	end
	return str
end
```

# table
```
tableIndex = findTableIndex("duty")

TurbochargerSpeed = getSensor("TurbochargerSpeed")
tps = getSensor("Tps1")

dutyCycle = table3d(tableIndex, TurbochargerSpeed, tps)

```


## See also

[Lua Ternary Operator](http://lua-users.org/wiki/TernaryOperator)
