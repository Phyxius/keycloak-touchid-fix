# Keycloak TouchID/FaceID Fix

This is a custom theme for the [Keycloak][keycloak] identity provider that fixes it to comply with the WebKit anti-abuse measures that disallow biometrics from being used for WebAuthn if the credential request does not come from a user gesture (e.g., a button click/tap). The default Keycloak theme requests credentials on page load, which fails this requirement and makes it incompatible with Apple platform authenticators. This theme is a slight modification of the default one that attaches the credential request to a button instead. Note that this is currently not the only change required to make Keycloak work with TouchID/FaceID; you also need to hack around the counter verification value using a DBMS trigger. 