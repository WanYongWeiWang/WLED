# LumenCache usermod

This usermod will provide the LumenCache integration for SIB-EP-WLED.

The current scaffold only verifies that the module is registered and initialized. It does not yet implement UART transport, FQM parsing, Zone or Scene handling, or WLED state translation.

When loaded, `/json/info` includes a `Lumencache` entry with the value `loaded`.

## Build

Add `A_Lumencache` to `custom_usermods` in a local PlatformIO override environment, then compile that environment.
