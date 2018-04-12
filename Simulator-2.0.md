# Major Design Changes

## Robot Code as a Plugin

### The Current Approach

The robot code runs as a standalone executable. It communicates with the simulator via [Ignition Transport](http://ignition-transport.readthedocs.io), where debug information, sensor readings, motor commands, and time are send and received.

### The Proposed Approach

Robot code will be compiled into a shared library and linked with the simulator libraries. The simulator will load and run the code via `dlopen`. Robot code will communicate with the simulator via overloaded methods inherited from some `smartmouse::Plugin` based class.

#### Advantages

 - Testability. The robot can can actually have tests now because it will be run within the simulator. The tests for the simulator can also be more robust because there is no asynchronous message passing
 - Debuggability. The simulator and the robot code run in the same process in lock-step, so when you hit a breakpoint to inspect something, time stops and the simulator is halted as well.
 - Determinism. Message passing introduce asnchrony, and therefore non-determinism, which is bad.
 - Fewer dependencies. We no longer need ignition transport, math, utils, messages, etc...

#### Disadvantages

 - Tighter coupling between robot code and simulator, since they must be API & ABI compatible. Previously they only needed to have compatible protobuf messages.
 - More code. The plugin API will make a lot of new code for dynamically loading and unloading shared libraries

## Mocking Embedded libraries

### The Current Approach

Code that includes any Teensyduino or Arduino code is seperated, and not compiled in simulation. Similarly, there is a robot code compiled only for simulation. This is why there are a `real/commands` and a `sim/commands` folder with the same classes with _slightly_ different implementation. The main difference in this code is just the use of LEDs and SPI in the real code and Ignition Transport in the simulation code. There is is also a seperate implementation for ASCII-only maze solving, which uses neither of these.

### The Proposed Approach

Completely remove the `console` implementation. It's original purpose was to test maze solving algorithms, but that we never use it anymore. Plus, maze solving is currently unit-tested without any reference to `console/commands`.

Create a hardware abstraction layer (HAL) that has the same API/ABI as the teensy, arduino, or whatever emebedded platform libraries one is using. For example, there will be a `digitalWrite` function, as well as a `Wire` class with a `transaction` method. The simulation and real commands will be merged. Here is a list of all the functions that would need to be mocked given the current real robot code:

    # TODO: make a list here so we know what we're getting in to

#### Advantages

 - Fewer bugs. Code duplication leads to copy/paste errors and bugs cause by slight implementation differences
 - Less Confusion. Code duplicaiton is confusing.
 - Better Abstractions. Hopefully by forcing a common API for simulation and real robot we will find a nice level of abstraction between hardware and software.

#### Disadvantages

 - HAL will be confusing. There will have to either be exceptions or empty function for things which are not implemented. What happens if someone calls `digitalWrite` to a pin that the simulated robot doesn't know what to do with?
 - Tightly coupled API. You will have to constantly change the HAL in order to support new hardware. If you switch from Arduino to MBed, for example, you would need a lot of changes in HAL, or you would need different HALs for different platforms.
