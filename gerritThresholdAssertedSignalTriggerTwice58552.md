RFC:ThresholdAsserted signal trigger twice

When the modified threshold is lower than the actual value,
the sensor will trigger the ThresholdAsserted signal twice.

Test process：
```shell
~# ipmitool sensor  get Inlet_Temp
Locating sensor record...
Sensor ID              : Inlet_Temp (0x1)
 Entity ID             : 7.0
 Sensor Type (Threshold)  : Temperature
 Sensor Reading        : 40 (+/- 0) degrees C
 Status                : ok
 Lower Non-Recoverable : na
 Lower Critical        : 0.000
 Lower Non-Critical    : 0.000
 Upper Non-Critical    : 42.000
 Upper Critical        : 47.000
 Upper Non-Recoverable : na
 Positive Hysteresis   : Unspecified
 Negative Hysteresis   : Unspecified

~# ipmitool sensor thresh Inlet_Temp unc 39
Locating sensor record 'Inlet_Temp'...
Setting sensor "Inlet_Temp" Upper Non-Critical threshold to 39.000

~# ipmitool sensor get Inlet_Temp
Locating sensor record...
Sensor ID              : Inlet_Temp (0x1)
 Entity ID             : 7.0
 Sensor Type (Threshold)  : Temperature
 Sensor Reading        : 40 (+/- 0) degrees C
 Status                : Upper Non-Critical
 Lower Non-Recoverable : na
 Lower Critical        : 0.000
 Lower Non-Critical    : 0.000
 Upper Non-Critical    : 39.000
 Upper Critical        : 47.000
 Upper Non-Recoverable : na
 Positive Hysteresis   : Unspecified
 Negative Hysteresis   : Unspecified

~# dbus-monitor --system --monitor "member=ThresholdAsserted"
signal time=1945.549071 sender=org.freedesktop.DBus -> destination=:1.699 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameAcquired
   string ":1.699"
signal time=1945.551366 sender=org.freedesktop.DBus -> destination=:1.699 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameLost
   string ":1.699"
signal time=1953.749817 sender=:1.75 -> destination=(null destination) serial=698 path=/xyz/openbmc_project/sensors/temperature/Inlet_Temp; interface=xyz.openbmc_project.Sensor.Threshold.Warning; member=ThresholdAsserted
   string "Inlet_Temp"
   string "xyz.openbmc_project.Sensor.Threshold.Warning"
   string "WarningAlarmHigh"
   boolean true
   double 40
signal time=1954.464875 sender=:1.75 -> destination=(null destination) serial=729 path=/xyz/openbmc_project/sensors/temperature/Inlet_Temp; interface=xyz.openbmc_project.Sensor.Threshold.Warning; member=ThresholdAsserted
   string "Inlet_Temp"
   string "xyz.openbmc_project.Sensor.Threshold.Warning"
   string "WarningAlarmHigh"
   boolean true
   double 40
```

```
  EntityManager               |              Dbus-sensors
+--------------------------+  |
|      INIT                |  |
|   Thresholds1.Value=42   |  |             +--------------------+
|                          |  |     +------>|matchPropertyChange |
| send PropertyChange -----+--+-----+       +--------+-----------+
+--------------------------+  |                      |
                              |    +----------------------------------------+
                              |    |   CREATE SENSOR                        |
                              |    | currentSensorReading=40                |
                              |    | alarm=false                            |
                              |    | check (SensorReading=40)<(Threshold=42)|
                              |    | alarm=false                            |
                              |    +----------------------------------------+
                              |
                              |  +-----------------------------------------+
                              |  |  Threshold=39     <---------------------+-----ipmitool sensor thresh Inlet_Temp unc 39
                              |  | Check (SensorReading=40)>(Threshold=39) |
                              |  | alarm=true      ------------------------+-----> The first signal WarningAlarmHigh
                              |  +-----------------------------------------+
+----------------------+      |
| hresholds1.Value=39  | <----+---sync Threshold to EntityManager
|                      |      |         +--------------------+
| send PropertyChange  |------+-------->| matchPropertyChange|
+----------------------+      |         +--------------------+
                              |                   |
                              |  +------------------------------------+
                              |  |  CREATE SENSOR                     |
                              |  | alarm=false                        |
                              |  | check SensorReading>Threshold=39   |
                              |  | alarm=true       -----------------+-------> The second signal WarningAlarmHigh
                              |  +------------------------------------+
```
If only the signal transmission of Threshold Value is canceled, the Dbus sensor creation will not be affected,
and the secondary creation of the Dbus sensors when the synchronization threshold to EM will be prevented.

My personal approach is remove the emits-change in FLAGS of Threshold Value
Before:
```
root@NULL:~# busctl  introspect  xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/Inlet_Temp
NAME                                                 TYPE      SIGNATURE RESULT/VALUE         FLAGS
...
xyz.openbmc_project.Configuration.TMP112.Thresholds1 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper non critical" emits-change writable
.Severity                                            property  d         0                    emits-change writable
.Value                                               property  d         42                   emits-change writable
root@NULL:~#
```
After this modification:
```
root@NULL:~# busctl  introspect  xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/Inlet_Temp
NAME                                                 TYPE      SIGNATURE RESULT/VALUE         FLAGS
...
xyz.openbmc_project.Configuration.TMP112.Thresholds1 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper non critical" emits-change writable
.Severity                                            property  d         0                    emits-change writable
.Value                                               property  d         42                   writable
root@NULL:~#
```

Code process：

~### Step 1: ipmi cmd
getService:
https://github.com/openbmc/phosphor-host-ipmid/blob/master/sensorhandler.cpp#L876

```
root@NULL:~# busctl call  --verbose   xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/object_mapper  xyz.openbmc_project.ObjectMapper GetObject   sas  /xyz/openbmc_project/sensors/temperature/Inlet_Temp 3 xyz.openbmc_project.Sensor.Value  xyz.openbmc_project.Sensor.Threshold.Warning xyz.openbmc_project.Sensor.Threshold.Critical
MESSAGE "a{sas}" {
        ARRAY "{sas}" {
                DICT_ENTRY "sas" {
                        STRING "xyz.openbmc_project.HwmonTempSensor";
                        ARRAY "s" {
                                STRING "xyz.openbmc_project.Association.Definitions";
                                STRING "xyz.openbmc_project.Sensor.Threshold.Critical";
                                STRING "xyz.openbmc_project.Sensor.Threshold.Warning";
                                STRING "xyz.openbmc_project.Sensor.Value";
                                STRING "xyz.openbmc_project.State.Decorator.Availability";
                                STRING "xyz.openbmc_project.State.Decorator.OperationalStatus";
                        };
                };
        };
};

```

~### Step 2: ipmi cmd
getAllDbusProperties:
https://github.com/openbmc/phosphor-host-ipmid/blob/master/sensorhandler.cpp#L893
```
root@NULL:~# busctl  call --verbose xyz.openbmc_project.HwmonTempSensor /xyz/openbmc_project/sensors/temperature/Inlet_Temp org.freedesktop.DBus.Properties GetAll s xyz.openbmc_project.Sensor.Threshold.Warning
MESSAGE "a{sv}" {
        ARRAY "{sv}" {
                DICT_ENTRY "sv" {
                        STRING "WarningHigh";
                        VARIANT "d" {
                                DOUBLE 42;
                        };
                };
                DICT_ENTRY "sv" {
                        STRING "WarningAlarmHigh";
                        VARIANT "b" {
                                BOOLEAN true;
                        };
                };
        };
};

root@NULL:~#
```

~### Step 3: ipmi cmd
setDbusProperty
https://github.com/openbmc/phosphor-host-ipmid/blob/master/sensorhandler.cpp#L957
```
root@NULL:~# busctl set-property   xyz.openbmc_project.HwmonTempSensor /xyz/openbmc_project/sensors/temperature/Inlet_Temp xyz.openbmc_project.Sensor.Threshold.Warning WarningHigh  d 39
```

~### Step 4: dbus-sensors
async_method_call:Set ,async to EmtityManager
https://github.com/openbmc/dbus-sensors/blob/master/src/Thresholds.cpp#L183
```
root@NULL:~# busctl set-property xyz.openbmc_project.EntityManager  /xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/SYS_5V  xyz.openbmc_project.Configuration.ADC.Thresholds0   Value  d  7.000
root@NULL:~#

root@NULL:~# dbus-monitor --system --monitor "member=PropertiesChanged,path_namespace=/xyz/openbmc_project/inventory"
signal time=161.145204 sender=org.freedesktop.DBus -> destination=:1.135 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameAcquired
   string ":1.135"
signal time=161.212438 sender=org.freedesktop.DBus -> destination=:1.135 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameLost
   string ":1.135"
signal time=167.907330 sender=:1.44 -> destination=(null destination) serial=2386 path=/xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/Inlet_Temp; interface=org.freedesktop.DBus.Properties; member=PropertiesChanged
   string "xyz.openbmc_project.Configuration.TMP112.Thresholds1"
   array [
      dict entry(
         string "Value"
         variant             double 39
      )
   ]
   array [
   ]
signal time=169.061588 sender=:1.30 -> destination=(null destination) serial=3011 path=/xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/all_sensors; interface=org.freedesktop.DBus.Properties; member=PropertiesChanged
   string "xyz.openbmc_project.Association"
   array [
      dict entry(
         string "endpoints"
         variant             array [
               string "/xyz/openbmc_project/sensors/power/Total_Power"
               string "/xyz/openbmc_project/sensors/temperature/PCH_Temp"
               string "/xyz/openbmc_project/sensors/voltage/SYS_12V"
               string "/xyz/openbmc_project/sensors/voltage/SYS_5V"
               string "/xyz/openbmc_project/sensors/voltage/RTC_Battery"
               string "/xyz/openbmc_project/sensors/voltage/SYS_3V3"
               string "/xyz/openbmc_project/control/Fan_Redundant"
               string "/xyz/openbmc_project/sensors/temperature/Outlet_Temp"
            ]
      )
   ]
   array [
   ]
signal time=169.064628 sender=:1.30 -> destination=(null destination) serial=3013 path=/xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/all_sensors; interface=org.freedesktop.DBus.Properties; member=PropertiesChanged
   string "xyz.openbmc_project.Association"
   array [
      dict entry(
         string "endpoints"
         variant             array [               string "/xyz/openbmc_project/sensors/power/Total_Power"
               string "/xyz/openbmc_project/sensors/temperature/PCH_Temp"
               string "/xyz/openbmc_project/sensors/voltage/SYS_12V"
               string "/xyz/openbmc_project/sensors/voltage/SYS_5V"
               string "/xyz/openbmc_project/sensors/voltage/RTC_Battery"
               string "/xyz/openbmc_project/sensors/voltage/SYS_3V3"
               string "/xyz/openbmc_project/control/Fan_Redundant"
               string "/xyz/openbmc_project/sensors/temperature/Outlet_Temp"
               string "/xyz/openbmc_project/sensors/temperature/Inlet_Temp"
            ]
      )
   ]
   array [
   ]
^C
root@NULL:~#
```

For the first time:ThresholdAsserted
After the first modification of the Threshold for dbus-sensors, when updating the Value, sensor need to signal_send("ThresholdAsserted")

~### Step 5:
Dbus sensors listen to PropertiesChanged, and then perform the CreateSensor operation：
https://github.com/openbmc/dbus-sensors/blob/master/src/HwmonTempMain.cpp#L680

CreateSensor will set the Alarm=false of init sensor;
https://github.com/openbmc/dbus-sensors/blob/master/include/sensor.hpp#L318

The second time:ThresholdAsserted
The sensor updateValue will do that signal_send("ThresholdAsserted")
So we have ThresholdAsserted twice.

The Property:
root@NULL:~# busctl  introspect  xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/Inlet_Temp
NAME                                                 TYPE      SIGNATURE RESULT/VALUE         FLAGS
org.freedesktop.DBus.Introspectable                  interface -         -                    -
.Introspect                                          method    -         s                    -
org.freedesktop.DBus.Peer                            interface -         -                    -
.GetMachineId                                        method    -         s                    -
.Ping                                                method    -         -                    -
org.freedesktop.DBus.Properties                      interface -         -                    -
.Get                                                 method    ss        v                    -
.GetAll                                              method    s         a{sv}                -
.Set                                                 method    ssv       -                    -
.PropertiesChanged                                   signal    sa{sv}as  -                    -
xyz.openbmc_project.Configuration.TMP112             interface -         -                    -
.Address                                             property  t         73                   emits-change
.Bus                                                 property  t         200                  emits-change
.Name                                                property  s         "Inlet_Temp"         emits-change
.Type                                                property  s         "TMP112"             emits-change
xyz.openbmc_project.Configuration.TMP112.Thresholds0 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper critical"     emits-change writable
.Severity                                            property  d         1                    emits-change writable
.Value                                               property  d         47                   emits-change writable
xyz.openbmc_project.Configuration.TMP112.Thresholds1 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper non critical" emits-change writable
.Severity                                            property  d         0                    emits-change writable
.Value                                               property  d         42                   emits-change writable
root@NULL:~#

After this modification: Value FLAGS: only writable
```
root@NULL:~# busctl  introspect  xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/FP5280G2_Motherboard/Inlet_Temp
NAME                                                 TYPE      SIGNATURE RESULT/VALUE         FLAGS
org.freedesktop.DBus.Introspectable                  interface -         -                    -
.Introspect                                          method    -         s                    -
org.freedesktop.DBus.Peer                            interface -         -                    -
.GetMachineId                                        method    -         s                    -
.Ping                                                method    -         -                    -
org.freedesktop.DBus.Properties                      interface -         -                    -
.Get                                                 method    ss        v                    -
.GetAll                                              method    s         a{sv}                -
.Set                                                 method    ssv       -                    -
.PropertiesChanged                                   signal    sa{sv}as  -                    -
xyz.openbmc_project.Configuration.TMP112             interface -         -                    -
.Address                                             property  t         73                   emits-change
.Bus                                                 property  t         200                  emits-change
.Name                                                property  s         "Inlet_Temp"         emits-change
.Type                                                property  s         "TMP112"             emits-change
xyz.openbmc_project.Configuration.TMP112.Thresholds0 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper critical"     emits-change writable
.Severity                                            property  d         1                    emits-change writable
.Value                                               property  d         47                   writable
xyz.openbmc_project.Configuration.TMP112.Thresholds1 interface -         -                    -
.Delete                                              method    -         -                    -
.Direction                                           property  s         "greater than"       emits-change writable
.Name                                                property  s         "upper non critical" emits-change writable
.Severity                                            property  d         0                    emits-change writable
.Value                                               property  d         42                   writable
root@NULL:~#
```

