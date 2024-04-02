---
title:  "Marvell CN9130 Temperature Regulation"
author: Tobias Waldekranz
date:   2024-04-02 18:54:00 +0100
tags:
 - linux
 - cn9130
 - thermal
---

Seeing as we are deploying the [CN9130][1] SoC from Marvell in
multiple products targeting industrial applications, we wanted to
explore how it reacts when exposed to high ambient temperatures.

Software wise we are running a more or less unmodified Linux 6.6.22
kernel, using the standard `performance` governor to manage CPU
frequency scaling.

_TL;DR: The system will will start throttling down the frequency of a
core, from the maximum of 1.6 GHz, to 800 MHz when it reaches 85°C. If
the core temperature exceeds 95°C, then the frequency is brought down
to 400 MHz. If either die on the chip reports a temperature in excess
of 100°C, an emergency shutdown is triggered to protect the hardware
from permanent damage._


## Passive Temperature Regulation

Looking at the relevant parts of the SoC's [device tree][2], we find
the following section:

```c
...
	thermal-zones {
		...
		ap_thermal_cpu0: ap-cpu0-thermal {
			polling-delay-passive = <1000>;
			polling-delay = <1000>;

			thermal-sensors = <&ap_thermal 1>;

			trips {
				cpu0_hot: cpu0-hot {
					temperature = <85000>;
					hysteresis = <2000>;
					type = "passive";
				};
				cpu0_emerg: cpu0-emerg {
					temperature = <95000>;
					hysteresis = <2000>;
					type = "passive";
				};
			};

			cooling-maps {
				map0_hot: map0-hot {
					trip = <&cpu0_hot>;
					cooling-device = <&cpu0 1 2>,
						<&cpu1 1 2>;
				};
				map0_emerg: map0-ermerg {
					trip = <&cpu0_emerg>;
					cooling-device = <&cpu0 3 3>,
						<&cpu1 3 3>;
				};
			};
		};
		...
	};
...
```

This defines the policy of how the kernel's [thermal][3] subsystem is
allowed to throttle the CPU frequency of a core in response to high
temperature environments. First, two trip points are defined:

- `cpu0-hot` at 85°C
- `cpu0-emerg` at 95°C

These serve as triggers for enforcing different CPU frequency
limitations via the set of defined `cooling-maps`. Deciphering this,
we see that:

- When `cpu0-hot` trips, i.e. core 0's temperature reaches 85°C, then
  we enable two cooling devices (`cpu0` and `cpu1`). At this point we
  require a cooling state in the interval from 1 to 2 (higher values
  meaning more aggressive cooling). `cpu0` and `cpu1` provides this
  cooling, not by enabling a fan, but by scaling down their core
  frequencies. Looking at the divider values in the `cpufreq`
  [driver][4], we see that state 1 corresponds to 800 MHz and state 2
  to 533 MHz.

- Correspondingly, when `cpu0-emerg` trips at 95°C, we require the
  most aggressive cooling state, 3, which corresponds to a core
  frequency of 400 MHz.

Equivalent policy declarations exist for the other three CPU
cores. Notice how the frequency throttling is applied to a pair of
cores: a high temperature on core 0 will limit both its own and its
neighboring core's (1) frequency. The same goes for cores 2 and 3.

If your kernel is built with `CONFIG_THERMAL_EMULATION` enabled, you
can verify the operation of these policies by artificially raising the
temperature reported by a given sensor. In this scenario, the system
is running in a room temperature setting, with all cores reporting
temperatures around 45°C:

```
root@infix:/sys/class/thermal$ cat thermal_zone[1234]/temp
45519
45519
45519
45519
```

Given that the system is managed by the `performance` governor, all
cores are running at the maximum frequency:

```
root@infix:/sys/bus/cpu/devices$ cat cpu*/cpufreq/cpuinfo_cur_freq
1600000
1600000
1600000
1600000
```

By emulating a "hot" temperature on core 1, and an "emergency"
temperature on core 2, we can see that the kernel enforces the
expected cooling policies:

```
root@infix:/sys/class/thermal$ echo 88888 >thermal_zone2/emul_temp
root@infix:/sys/class/thermal$ echo 99999 >thermal_zone3/emul_temp
root@infix:/sys/class/thermal$ cd ../../bus/cpu/devices/
root@infix:/sys/bus/cpu/devices$ cat cpu*/cpufreq/cpuinfo_cur_freq
800000
800000
400000
400000
```


## Critical Temperature Protection

In the event that passive cooling measures are not enough to keep the
chip below the maximum rated junction temperature, critical trip
points are defined, which when triggered unconditionally shuts down
the system in a final attempt to protect the hardware from permanent
damage.

The CN9130 is a chiplet design made up of two dies:

- The application processor die ("AP")
- The I/O co-processor die ("CP")

Each die is setup to trigger an interrupt if a temperature above 100°C
is detected, at which point the kernel will initiate a controlled
shutdown of the system.

Here is the relevant [device tree][5] section for the AP trip point:

```c
...
	thermal-zones {
		ap_thermal_ic: ap-ic-thermal {
			polling-delay-passive = <0>; /* Interrupt driven */
			polling-delay = <0>; /* Interrupt driven */

			thermal-sensors = <&ap_thermal 0>;

			trips {
				ap_crit: ap-crit {
					temperature = <100000>; /* mC degrees */
					hysteresis = <2000>; /* mC degrees */
					type = "critical";
				};
			};

			cooling-maps { };
		};
		...
	};
...
```

And [correspondingly][6] for the CP:

```c
...
	thermal-zones {
		CP11X_LABEL(thermal_ic): CP11X_NODE_NAME(ic-thermal) {
			polling-delay-passive = <0>; /* Interrupt driven */
			polling-delay = <0>; /* Interrupt driven */

			thermal-sensors = <&CP11X_LABEL(thermal) 0>;

			trips {
				CP11X_LABEL(crit): crit {
					temperature = <100000>; /* mC degrees */
					hysteresis = <2000>; /* mC degrees */
					type = "critical";
				};
			};

			cooling-maps { };
		};
	};
...
```

Just as with the passive cooling policy, we can verify the critical
temperature trip points by injecting a higher value via the emulation
interface:

```
root@infix:/sys/class/thermal$ echo 105000 >thermal_zone0/emul_temp
[443523.262143] thermal thermal_zone0: ap-ic-thermal: critical temperature reached, shutting down
[443523.270852] reboot: HARDWARE PROTECTION shutdown (Temperature too high)

Message from syslogd@infix at Tue Feb 23 17:28:00 2038 ...
kernel: thermal thermal_zone0: ap-ic-thermal: critical temperature reached, shutting down

Message from syslogd@infix at Tue Feb 23 17:28:00 2038 ...
kernel: reboot: HARDWARE PROTECTION shutdown (Temperature too high)
[ OK ] Saving system time (UTC) to RTC
[ OK ] Saving random seed
[ OK ] Stopping Device event daemon (udev)
[ OK ] Stopping D-Bus message bus daemon
[ OK ] Stopping Configuration daemon
[ OK ] Stopping NETCONF server
[ OK ] Stopping DHCP/DNS proxy
[ OK ] Stopping Container job runner
[ OK ] Stopping CLI backend daemon
[ OK ] Stopping Software update service
[ OK ] Stopping OpenSSH daemon
[ OK ] Stopping Status daemon
[ OK ] Stopping Zebra routing daemon
[ OK ] Stopping LLDP daemon (IEEE 802.1ab)
[ OK ] Stopping Chrony NTP v3/v4 daemon
[ OK ] Unmounting filesystems ...
[ OK ] Calling hook/svc/down scripts ...
[ OK ] Powering down ...
[443526.788409] reboot: Power down
```

[1]: https://www.marvell.com/content/dam/marvell/en/public-collateral/embedded-processors/marvell-infrastructure-processors-octeon-tx2-cn913x-product-brief.pdf
[2]: https://elixir.bootlin.com/linux/v6.6.22/source/arch/arm64/boot/dts/marvell/armada-ap80x.dtsi#L337
[3]: https://docs.kernel.org/driver-api/thermal/index.html
[4]: https://elixir.bootlin.com/linux/v6.6.22/source/drivers/cpufreq/armada-8k-cpufreq.c#L35
[5]: https://elixir.bootlin.com/linux/v6.6.22/source/arch/arm64/boot/dts/marvell/armada-ap80x.dtsi#L320
[6]: https://elixir.bootlin.com/linux/v6.6.22/source/arch/arm64/boot/dts/marvell/armada-cp11x.dtsi#L28
