# Extensions Messaging (`extensionsmessaging`)

The `extensionsmessaging` module is a standalone Android library (`minSdkVersion 29`) that helps Android apps implementing a browser extension native messaging host service (`IBrowserNativeMessageService`) verify the identity of the calling browser and the verification status of the connecting extension.

It has no runtime dependency on `:androidbrowserhelper` (Trusted Web Activities).

## Setup

1. Enable AIDL generation and add the dependency to your module's `build.gradle`:

```groovy
android {
    buildFeatures {
        aidl = true
    }
}

dependencies {
    implementation 'com.google.androidbrowserhelper:extensionsmessaging:1.0.0'
}
```

2. Copy the Chromium extension native messaging `.aidl` files (`IBrowserNativeMessageService.aidl`, `IConnectExtensionCallback.aidl`, `IExtensionNativeMessageService.aidl`, `IConnectPortCallback.aidl`, `IExtensionNativeMessagePort.aidl`, `IExtensionNativeMessageCallback.aidl`, and `MessagePayload.aidl`) into your module under `src/main/aidl/org/chromium/chrome/browser/extensions/api/messaging/`.

3. Declare your native messaging `Service` in `AndroidManifest.xml`:

```xml
<service
    android:name=".MyNativeMessagingService"
    android:exported="true">
    <intent-filter>
        <action android:name="org.chromium.chrome.browser.extensions.messaging.action.NATIVE_MESSAGING" />
    </intent-filter>
</service>
```

> **Note:** On Android 11+ (API 30+), Android automatically grants a bound `Service` visibility into the calling client's package for the duration of `bindService()`, so no `<queries>` declaration is required in `AndroidManifest.xml` solely to verify an incoming caller.

## Usage

### Verifying Official Chrome Callers and Extensions

Call `BrowserIdentityVerifier.isTrustedChromeCaller(context)` **synchronously on the incoming Binder thread** inside `IBrowserNativeMessageService.Stub#connectExtension` (before calling `Binder.clearCallingIdentity()` or posting work to another thread/executor), verify that the extension is store/policy verified via `isVerifiedExtension(extensionInfo)`, and check `extensionId` against your app's allowlist:

```java
import android.app.Service;
import android.content.Intent;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;

import com.google.androidbrowserhelper.extensions.messaging.BrowserIdentityVerifier;

import org.chromium.chrome.browser.extensions.api.messaging.IBrowserNativeMessageService;
import org.chromium.chrome.browser.extensions.api.messaging.IConnectExtensionCallback;

import java.util.Collections;
import java.util.Set;

public class MyNativeMessagingService extends Service {
    private static final Set<String> ALLOWED_EXTENSION_IDS =
            Collections.singleton("abcdefghijklmnopabcdefghijklmnop");

    private final IBrowserNativeMessageService.Stub mBinder =
            new IBrowserNativeMessageService.Stub() {
                @Override
                public void connectExtension(
                        String extensionId,
                        Bundle extensionInfo,
                        IConnectExtensionCallback callback) {
                    if (callback == null) {
                        return;
                    }

                    // 1. Verify the Binder call comes from an official Google-signed Chrome
                    //    release (Stable, Beta, Dev, or Canary). Must run synchronously on the
                    //    incoming Binder thread before hopping to any executor/coroutine.
                    if (!BrowserIdentityVerifier.isTrustedChromeCaller(
                            getApplicationContext())) {
                        notifyError(callback, "Untrusted browser caller");
                        return;
                    }

                    // 2. Reject unverified (e.g. unpacked developer-mode) extensions, which can
                    //    specify an arbitrary extension ID via their manifest key.
                    if (!BrowserIdentityVerifier.isVerifiedExtension(extensionInfo)) {
                        notifyError(callback, "Unverified extension");
                        return;
                    }

                    // 3. Verify the extension ID against your app's own allowlist.
                    if (!ALLOWED_EXTENSION_IDS.contains(extensionId)) {
                        notifyError(callback, "Extension not allowed");
                        return;
                    }

                    try {
                        callback.onSuccess(createExtensionSession(extensionId));
                    } catch (RemoteException ignored) {
                        // Browser process died before receiving the session binder.
                    }
                }
            };

    private static void notifyError(IConnectExtensionCallback callback, String message) {
        try {
            callback.onError(message);
        } catch (RemoteException ignored) {
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

### Verifying Other Browsers

To verify another Chromium-based browser against your own pinned 32-byte SHA-256 signing certificate digests (for example obtained via `apksigner verify --print-certs browser.apk`), check the calling package name, load its expected 32-byte digests, and call `BrowserIdentityVerifier.isCallerSignedBy(context, expectedCerts)` synchronously on the Binder thread:

```java
String[] packages = getPackageManager().getPackagesForUid(Binder.getCallingUid());
boolean trusted = false;
if (packages != null) {
    for (String pkg : packages) {
        List<byte[]> expectedCerts = TRUSTED_BROWSER_CERTS.get(pkg);
        if (expectedCerts != null
                && BrowserIdentityVerifier.isCallerSignedBy(
                        getApplicationContext(), expectedCerts)) {
            trusted = true;
            break;
        }
    }
}
if (!trusted) {
    notifyError(callback, "Untrusted browser caller");
    return;
}
```
