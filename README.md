# Keycloak TouchID/FaceID Fix

This is a custom theme for the [Keycloak][keycloak] identity provider that fixes it to comply with the WebKit anti-abuse measures that disallow biometrics from being used for WebAuthn if the credential request does not come from a user gesture (e.g., a button click/tap). The default Keycloak theme requests credentials on page load, which fails this requirement and makes it incompatible with Apple platform authenticators. This theme is a slight modification of the default one that attaches the credential request to a button instead. Note that this is currently not the only change required to make Keycloak work with TouchID/FaceID; you also need to hack around the counter verification value using a DBMS trigger. 

[keycloak]: https://www.keycloak.org/

More information about using this can be found [on my blog post][blog-post], which is also reproduced below:

[blog-post]: https://polansky.co/blog/hacking-keycloak-to-support-touchid-faceid-authentication/

[Keycloak][keycloak] is an open source identity broker that allows you to combine user credentials from different providers (such as Google OAuth, LDAP, GitLab, etc.) as well as locally-stored credentials into a single authentication provider that can integrate with downstream applications using either SAML2.0 or OpenID Connect. I use it in my home lab as a single sign on provider using local accounts (I gave a [talk][talk] about my setup at [BSides Orlando 2020][bsides], check it out if you're interested!). 

[keycloak]: https://www.keycloak.org/
[talk]: https://polansky.co/files/talks/BSides_Orlando_2020-Perimeterless_Homelabbing.pdf
[bsides]: https://2020.bsidesorlando.org/#/agenda?day=2&lang=en&sessionId=17525000000045156

My setup leverages local user accounts with [WebAuthn][webauthn] for password-less authentication. Users log in by providing their user name, then are prompted by their browser to authenticate using their device's platform authenticator (a built-in security key created using the device's hardware platform). The authentication process requires a biometric or PIN, then provides cryptographic proof to the web application (in this case, Keycloak) that the user is who they say they are, complete with an attestation chain proving the authenticator is legitimate, and that the keys are stored in a hardware-backed non-exportable store. This is an excellent user experience that provides MFA without making users bother with passwords or authenticator apps. Keycloak [supports][keycloak-webauthn-setup] this experience, and it works great with Windows and Android devices. However, Apple's implementation (used in Safari on MacOS and iOS) has two quirks that make it incompatible with Keycloak as of this writing. 

[webauthn]: https://webauthn.guide/

[keycloak-webauthn-setup]: https://www.keycloak.org/docs/latest/server_admin/#passwordless-webauthn-together-with-two-factor

The Safari (or rather, WebKit, the underlying browser engine) WebAuthn quirks are deliberate, and detailed in their [announcement blog post][webkit-blog-post] for the feature:

[webkit-blog-post]: https://webkit.org/blog/11312/meet-face-id-and-touch-id-for-the-web/

1. WebKit will only allow biometrics to be used for authentication if the authentication prompt is the result of a "user gesture." In other words, a page that attempts to use WebAuthn on page load will only prompt the user to use their security key, which is not what we want --- a security key is inconvenient and the user verification flow for them is much more awkward than the platform authenticator, plus the expense of actually buying the key. The stated reason for this is to prevent annoying users by constantly prompting them to authenticate (since a cryptographic identity is a pretty darn good tracking vector). 
2. The underlying authenticator code (implemented using the Apple Secure Enclave) never increments the signature counter value, leaving it at zero always. The counter is intended to allow detection of cloned authenticators; the intent is that an authenticator will increment the counter on each signature, and a service can track the counter value and fail the authentication if the counter from a given login attempt is lower than the last one it saw. The WebKit blog post recommends using the attestation blob provided by the authenticator to detect clones instead. 

Unfortunately, those quirks both effect Keycloak's WebAuthn implementation:

1. Keycloak's authentication flow uses individual pages for each stage of authentication, and the WebAuthn one issues the authentication request on page load rather than via a button. This triggers WebKit's anti-abuse policy and prevents biometry from being used.
2. Keycloak requires the signature counter to increase on each signature; sending a zero counter value every time will cause every signature after the first to fail with a cryptic error message. Checking the Keycloak log file will be more enlightening, and show an exception related to counter verification. 

I've [filed][bug-1] [bugs][bug-2] (free account required to view) about this to the Keycloak issue tracker, but in the mean time both of these problems can be worked around. 

[bug-1]: https://issues.redhat.com/browse/KEYCLOAK-16075

[bug-2]: https://issues.redhat.com/browse/KEYCLOAK-16091

# Customizing the Keycloak Login Flow

Keycloak provides a robust custom theme system that allows you to customize pretty much any part of the user-facing parts of the application, including the various login flow subpages. First, setup Keycloak to use WebAuthn for first and second factor authentication using their [guide][keycloak-webauthn-setup], then come back to this post. 

We'll use a custom theme that modifies the login page to require a "gesture" (specifically, tapping/clicking on a button) to trigger the WebAuthn API, which will satisfy the WebKit anti-abuse measures with as few changes as possible from the default theme. You can find the theme files on [my GitHub][theme-files-repo]. Download them, then put the `touchid-fix` folder into the Keycloak `themes/` directory. Then restart Keycloak, and in the admin console change the default login theme to the `touchid-fix` theme. Instead of it prompting for WebAuthn credentials on load, you'll see a button to press to login instead. It also adds a button to manually report a login failure, which makes it easier to switch to your backup login method if you're on a device that doesn't have WebAuthn setup. 

[theme-files-repo]: https://github.com/Phyxius/keycloak-touchid-fix

This isn't the only thing you'll have to do to make it compatible though. If you leave it here, you'll notice that every authentication attempt after the first fails, which is caused by the counter verification thinking you have a cloned authenticator (since the Apple authenticators never increment the counter). 

# Hacking Around the Counter Verification

While the previous workaround uses intended functionality to customize Keycloak, this second workaround... is a bit of a hack. Basically, when you login with an authenticator that reports a counter value of *n*, Keycloak stores the value *n+1*; the next time you authenticate, it errors if your new counter value is less than *n+1*. We're going to work around this by using DBMS triggers to reset the counter to zero whenever Keycloak updates it. 

## Disclaimer

**This is a hack.** It is not supported functionality, nor is it guaranteed to continue working in the future. It could cause your Keycloak instance to stop working, or introduce a security weakness (beyond the fact that you're disabling a security mechanism explicitly). While I've been running my install like this for a while with no negative effects as far as I can tell, I make no warranty whatsoever that this won't do something horrible to your install, or cause demons to spew forth from your server. Proceed at your own risk.

## The Actual Hack

You'll need to use a separate database for Keycloak rather than its built-in one. An example `docker-compose` file to do this can be found in the [Keycloak Containers repo][keycloak-container-example]. **Note that the example file uses a *very* old version of MySQL that you will have to upgrade for this to work.** I use the `mysql:latest` image, which is bad practice in production but fine for a lab. You need at least version 8.2. You can export your realm, switch to using an external database, then re-import it if you already have users or clients configured. You could also use a different RDBMS, but you'll have to adapt the trigger.

[keycloak-container-example]: https://github.com/keycloak/keycloak-containers/blob/master/docker-compose-examples/keycloak-mysql.yml

Once you've got Keycloak running using a MySQL database, you'll need to login to the database yourself using the MySQL client. If you use the compose file from above, it's `[sudo] docker-compose exec mysql bash`, then log in normally using the `mysql` binary with your password. Once you're at the prompt, select the Keycloak MySQL database (`use keycloak;`) and then run the following commands:

```sql
delimiter //
create trigger touchid_hack before update on CREDENTIAL
    for each row
    begin
    if NEW.TYPE = "webauthn-passwordless" then
    set NEW.CREDENTIAL_DATA = regexp_replace(NEW.CREDENTIAL_DATA, '"counter":[0-9]+', '"counter":0');
    END IF;
    END;//
```

This creates a trigger that intercepts writes to the table and replaces the counter value with zero before committing the data. The counter value will be read back on authentication, which since it's zero will always succeed the duplication check.

And that's it. You can now register and use Apple platform authenticators in Keycloak! You also need to set the WebAuthn policy to the appropriate values (ES256 signatures allowed, indirect or no attestation, platform authenticator, user verification not disallowed).