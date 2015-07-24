### Introduction ###
Introducing a SmartCard API in the Android platform enables security related services for application developers. Access to secure elements should be as open as possible - _the Android way_ - and not protected per default with certificate checks.

However, it needs to be ensured that access to sensitive services can still be controlled and/or protected to hinder malicious applications to interfere with the system.

For SIM cards it's up to the MNO whether card applications should be accessible or not whereas access to the Mobile Security Card could still be open for any developer thus the Secure Element itself is the best entity to define the access control.<br />


### Android Security ###
Android is a multi-process system, where each application (and parts of the system) runs in its own process. Most security between applications and the system is enforced at the process level via standard Linux facilities, such as user and group IDs that are assigned to applications. Additional finer-grained security features are provided via a _permission_ mechanism.

This mechanism enforces restrictions on the specific operations which a particular process can perform, and per-URI permissions for granting ad-hoc access to specific pieces of data.

For more details, see http://developer.android.com/guide/topics/security/security.html.


##### Permissions #####
The SmartCard API uses the Android permission scheme to protect access to secure elements. Therefore it defines a permission `org.simalliance.openmobileapi.SMARTCARD` that a client application must request in its manifest in order to obtain access to the API.

At install time, the user is asked whether or not the application should receive access to his or her secure element.

Access to the lower layer components such as the secure element specific terminal implementation will be protected with the standard Android mechanisms.

##### Root Access #####
The whole Android security scheme is based on standard UID/GID checks, therefore applocations with _root access_ can overcome the security mechanism.

For example, it is possible that a _root_ user replaces APKs or native system services with modified versions that contain tracers or hooks for other applications. This means that the security concept of the SmartCard API can be broken on rooted phones but this is not an issue of the API design but a general issue on Linux based systems where _root_ can do everything.<br />


### SmartCard API System Security ###

There are two main security concerns that the Smartcard API takes into account:
  * Applications should not be able to access lower level components: more specifically, a malicious application should not be able to bind to a terminal. Terminals should enforce this restriction by requiring the permission "org.simalliance.openmobileapi.BIND_TERMINAL" to any app that tries to bind to it. Since this is a system permission only held by the SmartcardService, no third-party app will be able to talk directly to the terminals. In regards to the lowest level components (e.g., RIL/modem, NFC Stack, etc.), Smartcard API is not responsible for controlling access to them, and instead it relies on Android's own security mechanisms.
  * Applications should not be able to impersonate the a "system terminal": system terminals are those whose name starts with SIM, eSE or SD. Given their high dependency with the hardware on the phone, these terminals should be compiled with the system. The goal is that a malicious app cannot expose a terminal that claims to be the, e.g., SIM terminal. SmartcardService enforces this behavior by checking that the system terminals hold the permission "org.simalliance.openmobileapi.SYSTEM_TERMINAL", which is a system-level permission.


![](https://cloud.githubusercontent.com/assets/11645011/6865711/f8b8acee-d470-11e4-882d-de786fe5f342.png)


### SmartCard API Access Control Scheme ###
The SmartcardService implements and relies on Global Platform's Secure Element Access Control specification to allow or deny access to applications.

See [AccessControlIntroduction](AccessControlIntroduction) for more details.
