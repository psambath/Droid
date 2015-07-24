# Introduction #

The SmartCard API is a reference implementation of the SIMalliance Open Mobile API specification that enables Android applications to communicate with Secure Elements, e.g. SIM card, embedded Secure Elements, Mobile Security Card or others.

The interface of the library is kept platform independent whereas the SmartcardService implementation is Android specific.

With SmartCard API version 1 release a proprietary interface was introduced to provide access to Secure Elements. Version 2 included the Open Mobile API interface parallel to the proprietary one. Further releases considered the proprietary interface as deprecated and version 2.3.0 completely removes the old one and only supports the Open Mobile API.

Previous versions extended the Android framework with SmartCard API functionality which caused difficulties for application developers as modified platform SDK libraries were required. Since version 2.1.2 the shared library approach was introduced which does not extend the official framework but provides a library extension instead. Since version 2.3.0 the shared library approach is the only supported concept for the integration.

The SmartCard API provides functionality to list and select the supported Secure Elements, open a communication channel to a dedicated Secure Element application and transfer APDUs to such. No applet selection or channel management APDU functions are allowed as this might cause security problems. Also, no card on/off functions are available in contrast to PC/SC.

## SmartCard API modules ##
The SmartCard API consists of several software layers:
  * <b>SmartcardService:</b> Android remote service which is the core of the smart card access. Integration in <code>packages/apps/SmartcardService/src</code>
  * <b>SEService:</b> wrapper classes to hide the service binding specifics and deal as the main interface for application developers. Integration in <code>packages/apps/SmartCardService/openmobileapi</code>
  * <b>xxxTerminal:</b> Implementation of a Secure Element specific terminal, e.g. <code>UiccTerminal</code>, <code>ASSDTerminal</code>, ... Integration in <code>packages/services/xxxTerminal</code>
  * <b>Access Control Enforcer:</b> Implementation of the Access Control Enforcer to control APDU access from Android applications, according to GP SEAC Integration in <code>packages/apps/SmartCardService/src/org/simalliance/openmobileapi/service/security</code>

![](https://cloud.githubusercontent.com/assets/11645011/6865131/124e176c-d46b-11e4-9565-3fba4aea60e3.png)




## Architecture ##
The core of the SmartCard API is encapsulated in a remote Android service. Having a single service instance instead of a pure framework library ensures that security checks (who is accessing the service) and resource management (free a logical channel if a client dies) can be guaranteed - even if a client application hangs or dies unexpectedly.

However, using an Android service for an API interface is not typical for smart card applications thus additional framework classes wrap the service specifics, e.g. Binding to a Remote Service object. No further logic is kept in the framework classes, clients are permitted to access the service interface directly (but need to deal with the Service specifics by themselves).

The framework classes around `SEService` are just wrapping the service specifics for application developers.

The card channel is used for actual card communication by transferring APDUs. All channel management for logical card channels is encapsulated by the channel objects.

## System integration ##
The `smartcard-api.patch` includes access to all Secure Elements currently introduced. However, not all phones have SD card slots or embedded Secure Elements. System integrators should remove the builtin terminals the platform does not support (e.g. removing ASSDTerminal on phones that have no SD card slot like Nexus S).

The SmartCard API will enumerate the terminals at run time which means that removing a xxxTerminal folder from <code>packages/services/xxxTerminal</code> and any reference to it in <code>build/target/product/core.mk</code> is enough to remove a specific SE.

In addition, new terminals can be added by means of installing an APK that contains an Android Service with some specific requirements. See [this page](AddonTerminal) for furher details.

## Using the SmartCard API ##
Download and apply the patch files according to [Building building the system](BuildingTheSystem)

Compile the Android platform, build and flash a [device]([UNMAINTAINED]-Devices) or start a new [emulator](EmulatorExtension) instance with SmartCard API support.

Start working on your own project with a quick start from [Using the SmartCard API](UsingSmartCardAPI).

Import the [samples](https://github.com/seek-for-android/) and run them on the phone or in the emulator. They provide a good starting point.

See SmartCard API [JavaDoc](http://seek-for-android.github.io/javadoc/latest) for further documentation.

