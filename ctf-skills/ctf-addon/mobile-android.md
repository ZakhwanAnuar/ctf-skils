# Mobile / Android — Exploitation Quick Reference

Upstream covers iOS RE and Android forensics. This is the APK attack flow.

## APK teardown — fixed order

```bash
# 1. Static source-level
jadx-gui app.apk            # java source — read first, fastest understanding
jadx app.apk -d app_src     # CLI version
# 2. Resources + smali
apktool d app.apk -o app_dec
# Hunt order:
grep -rn "http\|api\|key\|secret\|token\|password" app_src/ | grep -v "\.so"
cat app_dec/AndroidManifest.xml     # exported components, permissions, deep links
ls app_dec/assets/ app_dec/res/raw/ # bundled configs, keys, extra files
# 3. Native libs
file app_dec/lib/*/*.so              # arm64-v8a etc -> Ghidra
strings -n 8 lib.so | grep -iE 'key|flag|secret|http'
# 4. Firebase / API keys:
grep -rn "firebaseio\|googleapis\|AIza" app_src/
curl https://<project>.firebaseio.com/.json     # open DB check
```

Manifest red flags:
```
exported="true" activities/services/receivers -> intent attacks
android:allowBackup="true" -> adb backup extract
debuggable="true" -> run-as com.app / attach debugger
deep links (scheme) -> web-to-app intent injection
```

## Network capture

```bash
# Option A — Burp: set proxy, install CA:
#   Android 7+: user CA not trusted by default -> bypass with objection (below)
#   or add res/xml/network_security_config.xml patch + rebuild + sign
# Option B — tcpdump on rooted device:
adb shell tcpdump -i any -w /sdcard/capture.pcap
# Option C — mitmproxy for scripted flows:
mitmproxy -p 8080 --set block_global=false
```

## Frida cookbook

```bash
pip install frida-tools
frida-ls-devices ; frida-ps -U                  # list processes on device
frida -U -f com.target.app -l hook.js --no-pause
```

SSL pinning bypass (universal):
```javascript
Java.perform(function () {
    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    var TrustManager = Java.registerClass({
        name: 'com.anon.TM', implements: [X509TrustManager],
        methods: { checkClientTrusted: function(){}, checkServerTrusted: function(){}, getAcceptedIssuers: function(){return [];} }
    });
    SSLContext.init.overload('[Ljavax.net.ssl.KeyManager;', '[Ljavax.net.ssl.TrustManager;', 'java.security.SecureRandom')
        .implementation = function (a, b, c) { SSLContext.init.overload(...)(a, [TrustManager.$new()], c); };
});
```
Or just: `objection -g com.app explore` then `android sslpinning disable`.

Root detection bypass: `android root disable` in objection.

Dump strings/secrets at runtime:
```javascript
Java.perform(function () {
    var SecretClass = Java.use('com.target.Secrets');
    console.log(SecretClass.getKey());
    // hook a check function to always return true:
    var Check = Java.use('com.target.LicenseCheck');
    Check.isValid.implementation = function () { return true; };
});
```

Method tracing (find what to hook without reading all source):
```javascript
Java.enumerateLoadedClasses({ onMatch: function(c){ if(c.includes('target')) console.log(c); }, onComplete:function(){} });
// or trace all methods of a class:
Java.use('com.target.C').class.getDeclaredMethods().forEach(function(m){ console.log(m); });
```

## objection one-liners

```bash
objection -g com.app explore
android sslpinning disable
android root disable
android hooking list classes / list class_methods com.target.C
android hooking watch class com.target.C              # all methods
android hooking watch class_method com.target.C.check --dump-args --dump-return
android intent launch_activity com.target.FlagActivity
memory list modules ; memory search "flag{"
```

## Rebuild + sign (after patching smali)

```bash
apktool b app_dec -o patched.apk
# sign (any debug key):
keytool -genkey -v -keystore ks.jks -alias a -keyalg RSA -keysize 2048 -validity 10000
apksigner sign --ks ks.jks patched.apk
zipalign -v 4 patched.apk patched-aligned.apk
# install
adb install patched-aligned.apk
```

## Common Android CTF bugs

```
- Hardcoded keys/secrets in strings.xml, BuildConfig, assets
- Weak crypto (ECB, static IV, key derived from IMEI/username)
- Exported activity/service: adb shell am start -n com.app/.FlagActivity
- Intent extras injection: craft extras to unlock premium/flag path
- WebView addJavascriptInterface (older) -> JS bridge RCE
- Local SQLite flag storage: adb pull /data/data/com.app/databases/
- WebView file:// access + XSS -> read local files
- Custom scheme handler reflected into WebView load -> local file theft
- Backup extraction: adb backup -apk com.app -> restore + abe.jar unpack
```
