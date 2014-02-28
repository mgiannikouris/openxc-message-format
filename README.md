# OpenXC Message Format Specification

This specification is a part of the [OpenXC platform][OpenXC].

An OpenXC vehicle interface sends generic vehicle data over one or more output
interfaces (e.g. USB or Bluetooth) as JSON objects, separated by newlines.

There are two valid message types - single valued and evented.

There may not be a 1:1 relationship between input and output signals - i.e. raw
engine timing CAN signals may be summarized in an "engine performance" metric on
the abstract side of the interface.

## Single Valued

The expected format of a single valued message is:

    {"name": "steering_wheel_angle", "value": 45}

## Evented

The expected format of an event message is:

    {"name": "button_event", "value": "up", "event": "pressed"}

This format is good for something like a button event, where there are two
discrete pieces of information in the measurement.

## Raw CAN Message format

An OpenXC vehicle interface may also output raw CAN messages. Each CAN message
is sent as a JSON object, separated by newlines. The format of each object is:

    {"bus": 1, "id": 1234, "value": "0x12345678"}

**bus** - the numerical identifier of the CAN bus where this message originated,
  most likely 1 or 2 (for a vehicle interface with 2 CAN controllers).

**id** - the CAN message ID

**data** - up to 8 bytes of data from the CAN message's payload, represented as
  a hexidecimal number in a string. Many JSON parser cannot handle 64-bit
  integers, which is why we are not using a numerical data type.

## Diagnostic Messages

### Requests

    {"bus": 1,
      "id": 1234,
      "mode": 1,
      "pid": 5,
      "payload": "0x1234",
      "parse_payload": true,
      "factor": 1.0,
      "offset": 0,
      "frequency": 0}

**bus** - the numerical identifier of the CAN bus where this request should be
    sent, most likely 1 or 2 (for a vehicle interface with 2 CAN controllers).

**id** - the CAN arbitration ID for the request.

**mode** - the OBD-II mode of the request - 1 through 15 (1 through 9 are the
    standardized modes).

**pid** - (optional) the PID for the request, if applicable.

**payload** - (optional) up to 7 bytes of data for the request's payload
    represented as a hexidecimal number in a string. Many JSON parser cannot
    handle 64-bit integers, which is why we are not using a numerical data type.

**parse_payload** - (optional, false by default) if true, the complete payload in the
    response message will be parsed as a number and returned in the 'value' field of
    the response. The 'payload' field will be omitted in responses with a
    'value'.

**factor** - (optional, 1.0 by default) if `parse_payload` is true, the value in
    the payload will be multiplied by this factor before returning. The `factor`
    is applied before the `offset`.

**offset** - (optional, 0 by default) if `parse_payload` is true, this offset
    will be added to the value in the payload before returning. The `offset` is
    applied after the `factor`.

**frequency** - (optional, defaults to 0) The frequency in Hz to send this
    request. To send a single request, set this to 0 or leave it out.

TODO it'd be nice to have the OBD-II PIDs built in, with the proper conversion
functions - that may need a different output format

If you're just requesting a PID, you can use a simplified format for the
request:

    {"bus": 1, "id": 1234, "mode": 1, "pid": 5}

### Responses

    {"bus": 1,
      "id": 1234,
      "mode": 1,
      "pid": 5,
      "success": true,
      "negative_response_code": 17,
      "payload": "0x1234",
      "parsed_payload": 4660}

**bus** - the numerical identifier of the CAN bus where this response was
    received.

**id** - the CAN arbitration ID for this response.

**mode** - the OBD-II mode of the original diagnostic request.

**pid** - (optional) the PID for the request, if applicable.

**success** -  true if the response received was a positive response. If this
  field is false, the remote node returned an error and the
  `negative_response_code` field should be populated.

**negative_response_code** - (optional)  If requested node returned an error,
    `success` will be `false` and this field will contain the negative response
    code (NRC).

Finally, the `payload` and `value` fields are mutually exclusive:

**payload** - (optional) up to 7 bytes of data returned in the response,
    represented as a hexadecimal number in a string. Many JSON parser cannot
    handle 64-bit integers, which is why we are not using a numerical data type.

**value** - (optional) if the response had a payload, this may be the
    payload interpreted as an integer and transformed with a factor and offset
    provided with the request.

The response to a simple PID request would look like this:

    {"bus": 1, "id": 1234, "mode": 1, "pid": 5, "payload": "0x2"}

TODO again, it'd be nice to have the OBD-II PIDs built in, with the proper
conversion functions so the response here included the actual transformed value
of the pid and a human readable name

## Trace File Format

An OpenXC vehicle trace file is a plaintext file that contains JSON objects,
separated by newlines.

The first line may be a metadata object, although this is optional:

```
{"metadata": {
    "version": "v3.0",
    "vehicle_interface_id": "7ABF",
    "vehicle": {
        "make": "Ford",
        "model": "Mustang",
        "trim": "V6 Premium",
        "year": 2013
    },
    "description": "highway drive to work",
    "driver_name": "TJ Giuli",
    "vehicle_id": "17N1039247929"
}
```

The following lines are OpenXC messages with a `timestamp` field added, e.g.:

    {"timestamp": 1385133351.285525, "name": "steering_wheel_angle", "value": 45}

The timestamp is in [UNIX time](http://en.wikipedia.org/wiki/Unix_time)
(i.e. seconds since the UNIX epoch, 00:00:00 UTC, 1/1/1970).

## Official Signals

These signal names are a part of the OpenXC specification, although some
manufacturers may support custom message names.

* steering_wheel_angle
    * numerical, -600 to +600 degrees
    * 10Hz
* torque_at_transmission
    * numerical, -500 to 1500 Nm
    * 10Hz
* engine_speed
    * numerical, 0 to 16382 RPM
    * 10Hz
* vehicle_speed
    * numerical, 0 to 655 km/h (this will be positive even if going in reverse
      as it's not a velocity, although you can use the gear status to figure out
      direction)
    * 10Hz
* accelerator_pedal_position
    * percentage
    * 10Hz
* parking_brake_status
    * boolean, (true == brake engaged)
    * 1Hz, but sent immediately on change
* brake_pedal_status
    * boolean (True == pedal pressed)
    * 1Hz, but sent immediately on change
* transmission_gear_position
    * states: first, second, third, fourth, fifth, sixth, seventh, eighth,
      reverse, neutral
    * 1Hz, but sent immediately on change
* gear_lever_position
    * states: neutral, park, reverse, drive, sport, low, first, second, third,
      fourth, fifth, sixth
    * 1Hz, but sent immediately on change
* odometer
    * Numerical, km
        0 to 16777214.000 km, with about .2m resolution
    * 10Hz
* ignition_status
    * states: off, accessory, run, start
    * 1Hz, but sent immediately on change
* fuel_level
    * percentage
    * 2Hz
* fuel_consumed_since_restart
    * numerical, 0 - 4294967295.0 L (this goes to 0 every time the vehicle
      restarts, like a trip meter)
    * 10Hz
* door_status
    * Value is State: driver, passenger, rear_left, rear_right.
    * Event is boolean: true == ajar
    * 1Hz, but sent immediately on change
* headlamp_status
    * boolean, true is on
    * 1Hz, but sent immediately on change
* high_beam_status
    * boolean, true is on
    * 1Hz, but sent immediately on change
* windshield_wiper_status
    * boolean, true is on
    * 1Hz, but sent immediately on change
* latitude
    * numerical, -89.0 to 89.0 degrees with standard GPS accuracy
    * 1Hz
* longitude
    * numerical, -179.0 to 179.0 degrees with standard GPS accuracy
    * 1Hz

License
=======

Copyright (c) 2012-2013 Ford Motor Company

Licensed under the BSD license.

[OpenXC]: http://openxcplatform.com
