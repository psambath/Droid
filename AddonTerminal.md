# Introduction #

The SmartCard API provides the possibility to add additional terminals without re-flashing the system. The concept is supported within the _smartcard-api.patch_ system integration as well as the _MSC_ _SmartcardService_.

This concept allows a post installation of Secure Element terminals like a Bluetooth, USB smart card reader or any other Secure Element connected to the device.

Starting with SCAPI 4.0.0, we've completely redesigned the terminal interface. Now terminals run on their own context and are accessed from SmartcardService using standard Android IPC mechanisms.

# Requirements #
The installation of an Addon Terminal requires the following conditions:
  * The application contains an Android Service that listens to the "org.simalliance.openmobileapi.TERMINAL_DISCOVERY" intent.
  * This service implements the specified AIDL (see below).
  * This service requires the "org.simalliance.openmobileapi.BIND_TERMINAL" permission to bind to it. If it doesn't, SmartcardService will NOT enumerate it.
  * The type of the terminal is specified with the "android:label" attribute in the service declaration.
  * The type of the terminal is NOT "SIM", "eSE" or "SD" -- these types are reserved to the "system" terminals".
  * The package has to get registered before the SmartCard API process is running

# Sample #
The sample code referenced here can also be found in [this Git repository](https://github.com/seek-for-android/sample-add-on-terminal).
  * Using the Eclipse project wizard, create a new Android project

> _File -> New -> Project_

  * Select _Android Project_ and click _Next_
  * To create the project, fill in the required fields

> Project name: _PluginTerminal_  
> Application name: _PluginTerminalSample_  
> Package name: _your.organization.domain.SamplePluginTerminal_  
> Create Activity: _deselect_

  * Click _Finish_ to create the body of the (empty) Android application
  * Create a new class in the project that extends the Android _Service_ class.
  * Override the onBind() method so it returns an implementation of the ITerminalService.aidl interface.
  * In order for the project to compile you will need the files _ITerminalService.aidl_, _OpenLogicalChannelResponse.aidl/.java_ and _SmartcardError.aidl/.java_. It is recommended that you import them from the shared library jar. Alternatively, you can copy them from the source code (directory `./common/src/org/simalliance/openmobileapi/service` of the [SmartcardService project](https://github.com/seek-for-android/platform_packages_apps_SmartCardService)).
  * Install the app and reboot the device (in order to force SmartcardService to restart).
  * Run the _OpenMobileApiSample_ application, _PerformanceTester_ or similar to check the Addon Terminal is properly installed.

# Details #
The Git [PluginTerminal](https://github.com/seek-for-android/sample-add-on-terminal) sample can be used to study the functionality of the Addon Terminal implementation.
The PluginTerminal sample is a dummy terminal that uses a mock card.

## Access Control ##
An implementation of an Addon Terminal must consider some characteristics required for smooth cooperation with the Access Control Enforcer.
The Access Control Enforcer tries to select an Access Control Applet on the particular secure element in order to retrieve access rules.
It's not required that such an applet is present on the secure element, but in that case the Acces Control Enforcer expects the terminal return "null" and with the "SmartcardError" set with a `NoSuchElementException` (`error.set(new NoSuchElementException("Applet not found"))`).
This behaviour must be implemented in method `public OpenLogicalChannelResponse internalOpenLogicalChannel(byte[] aid, byte p2, SmartcardError error)` (see sample code).

Also the Addon Terminal should check the caller application to assert that the caller is entitled to use the Addon Terminal (see sample code).

## Log of sample Addon Terminal _DummyTerminal_ ##
The logcat output of the sample Addon Terminal _DummyTerminal_ after running the [OpenMobileApiSample](https://github.com/seek-for-android/open-mobile-api-sample) application will show
```
V/DummyTerminal(  819): internalConnect
V/DummyTerminal(  819): internalOpenLogicalChannel: AID = D2 76 00 01 18 AA FF FF 49 10 48 89 01
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalDisconnect
V/DummyTerminal(  819): internalConnect
V/DummyTerminal(  819): internalTransmit: 80 CA 9F 7F 00
V/DummyTerminal(  819): internalOpenLogicalChannel: AID = D2 76 00 01 18 AA FF FF 49 10 48 89 01
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalTransmit: 81 CA 9F 7F 00
V/DummyTerminal(  819): internalCloseLogicalChannel: 0
V/DummyTerminal(  819): internalCloseLogicalChannel: 1
V/DummyTerminal(  819): internalDisconnect
V/DummyTerminal(  819): internalConnect
V/DummyTerminal(  819): internalOpenLogicalChannel: AID = D2 76 00 01 18 AA FF FF 49 10 48 89 01
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalDisconnect
V/DummyTerminal(  819): internalConnect
V/DummyTerminal(  819): internalTransmit: 00 A4 04 00 08 A0 00 00 00 03 00 00 00 00
V/DummyTerminal(  819): internalTransmit: 80 CA 9F 7F 00
V/DummyTerminal(  819): internalOpenLogicalChannel: AID = D2 76 00 01 18 AA FF FF 49 10 48 89 01
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalOpenLogicalChannel: AID = A0 00 00 00 03 00 00 00
V/DummyTerminal(  819): internalOpenLogicalChannel: default applet
V/DummyTerminal(  819): internalTransmit: 81 CA 9F 7F 00
V/DummyTerminal(  819): internalTransmit: 00 A4 04 00 00
V/DummyTerminal(  819): internalTransmit: 00 A4 04 00 0D D2 76 00 01 18 AA FF FF 49 10 48 89 01 00
V/DummyTerminal(  819): internalCloseLogicalChannel: 0
V/DummyTerminal(  819): internalCloseLogicalChannel: 1
V/DummyTerminal(  819): internalDisconnect
```

## Methods to be implemented ##

**Note:** Since Android doesn't support exceptions accross processes, none of this methods should throw an exception. Instead, if an exception has to be thrown to the client application, the SmartcardError object can be used to pass it to the upper layers (`error.set(exeption)`).

* `OpenLogicalChannelResponse internalOpenLogicalChannel(in byte[] aid, in byte p2, out SmartcardError error)`: Implements the required logic to open a logical channel: send a MANAGE CHANNEL (Open) command and, if it succeeds, a SELECT by AID command. The select by AID shall use the P2 parameter specified in the method call, and the data field shall contain the AID.
* `void internalCloseLogicalChannel(int channelNumber, out SmartcardError error)`: Sends a MANAGE CHANNEL close command.
* `byte[] internalTransmit(in byte[] command, out SmartcardError error)`: Implements the terminal specific transmit operation.
* `byte[] getAtr()`: Returns the ATR of the connected card or null if the ATR is not available.
* `boolean isCardPresent()`: Returns whether the card is present or nor.
* `byte[] simIOExchange(in int fileID, in String filePath, in byte[] cmd, out SmartcardError error)`: Exchanges APDU (SELECT, READ/WRITE) to the  given EF by File ID and file path via iccIO.
* `String getSeStateChangedAction()`: Returns the Intent Action to which the SmartcardService shall register to detect changes in this Secure Element State (inserted/removed). If no intent is available, return an empty String.