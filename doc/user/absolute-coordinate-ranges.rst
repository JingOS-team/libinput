.. _absolute_coordinate_ranges:

==============================================================================
Coordinate ranges for absolute axes
==============================================================================

libinput requires that all touchpads provide a correct axis range and
resolution. These are used to enable or disable certain features or adapt
the interaction with the touchpad. For example, the software button area is
narrower on small touchpads to avoid reducing the interactive surface too
much. Likewise, palm detection works differently on small touchpads as palm
interference is less likely to happen.

Touchpads with incorrect axis ranges generate error messages
in the form:
<blockquote>
Axis 0x35 value 4000 is outside expected range [0, 3000]
</blockquote>

This error message indicates that the ABS_MT_POSITION_X axis (i.e. the x
axis) generated an event outside the expected range of 0-3000. In this case
the value was 4000.
This discrepancy between the coordinate range the kernels advertises vs.
what the touchpad sends can be the source of a number of perceived
bugs in libinput.

.. _absolute_coordinate_ranges_fix:

------------------------------------------------------------------------------
Measuring and fixing touchpad ranges
------------------------------------------------------------------------------

To fix the touchpad you need to:

#. measure the physical size of your touchpad in mm
#. run touchpad-edge-detector
#. trim the udev match rule to something sensible
#. replace the resolution with the calculated resolution based on physical settings
#. test locally
#. send a patch to the systemd project

Detailed explanations are below.

`libevdev <http://freedesktop.org/wiki/Software/libevdev/>`_ provides a tool
called **touchpad-edge-detector** that allows measuring the touchpad's input
ranges. Run the tool as root against the device node of your touchpad device
and repeatedly move a finger around the whole outside area of the
touchpad. Then control+c the process and note the output.
An example output is below:


::

     $> sudo touchpad-edge-detector /dev/input/event4
     Touchpad SynPS/2 Synaptics TouchPad on /dev/input/event4
     Move one finger around the touchpad to detect the actual edges
     Kernel says:	x [1024..3112], y [2024..4832]
     Touchpad sends:	x [2445..4252], y [3464..4071]

     Touchpad size as listed by the kernel: 49x66mm
     Calculate resolution as:
	x axis: 2088/<width in mm>
	y axis: 2808/<height in mm>

     Suggested udev rule:
     # <Laptop model description goes here>
     evdev:name:SynPS/2 Synaptics TouchPad:dmi:bvnLENOVO:bvrGJET72WW(2.22):bd02/21/2014:svnLENOVO:pn20ARS25701:pvrThinkPadT440s:rvnLENOVO:rn20ARS25701:rvrSDK0E50512STD:cvnLENOVO:ct10:cvrNotAvailable:*
      EVDEV_ABS_00=2445:4252:<x resolution>
      EVDEV_ABS_01=3464:4071:<y resolution>
      EVDEV_ABS_35=2445:4252:<x resolution>
      EVDEV_ABS_36=3464:4071:<y resolution>



Note the discrepancy between the coordinate range the kernels advertises vs.
what the touchpad sends.
To fix the advertised ranges, the udev rule should be taken and trimmed
before being sent to the `systemd project <https://github.com/systemd/systemd>`_.
An example commit can be found
`here <https://github.com/systemd/systemd/commit/26f667eac1c5e89b689aa0a1daef6a80f473e045>`_.

In most cases the match can and should be trimmed to the system vendor (svn)
and the product version (pvr), with everything else replaced by a wildcard
(*). In this case, a Lenovo T440s, a suitable match string would be:
::

     evdev:name:SynPS/2 Synaptics TouchPad:dmi:*svnLENOVO:*pvrThinkPadT440s*


.. note:: hwdb match strings only allow for alphanumeric ascii characters. Use a
	wildcard (* or ?, whichever appropriate) for special characters.

The actual axis overrides are in the form:

::

     # axis number=min:max:resolution
      EVDEV_ABS_00=2445:4252:42

or, if the range is correct but the resolution is wrong

::

     # axis number=::resolution
      EVDEV_ABS_00=::42


Note the leading single space. The axis numbers are in hex and can be found
in *linux/input-event-codes.h*. For touchpads ABS_X, ABS_Y,
ABS_MT_POSITION_X and ABS_MT_POSITION_Y are required.

.. note:: The touchpad's ranges and/or resolution should only be fixed when
	there is a significant discrepancy. A few units do not make a
	difference and a resolution that is off by 2 or less usually does
	not matter either.

Once a match and override rule has been found, follow the instructions at
the top of the
`60-evdev.hwdb <https://github.com/systemd/systemd/blob/master/hwdb/60-evdev.hwdb>`_
file to save it locally and trigger the udev hwdb reload. Rebooting is
always a good idea. If the match string is correct, the new properties will
show up in the
output of

::

        udevadm info /sys/class/input/event4


Adjust the command for the event node of your touchpad.
A udev builtin will apply the new axis ranges automatically.

When the axis override is confirmed to work, please submit it as a pull
request to the `systemd project <https://github.com/systemd/systemd>`_.
