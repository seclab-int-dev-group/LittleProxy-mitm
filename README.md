[![Build Status](https://travis-ci.org/ganskef/LittleProxy-mitm.png?branch=master)](https://travis-ci.org/ganskef/LittleProxy-mitm)
[![Published Version](https://img.shields.io/maven-central/v/com.github.ganskef/littleproxy-mitm.svg)](http://search.maven.org/#search|ga|1|g%3A%22com.github.ganskef%22%20AND%20a%3A%22littleproxy-mitm%22)

LittleProxy - Man-In-The-Middle
===============================

LittleProxy-mitm is an extension for 
[LittleProxy](https://github.com/adamfisk/LittleProxy) which provides all the 
filter capabilities of LittleProxy with HTTPS sites too. It aims to support 
every Java platform including Android. To answer HTTPS while offline for caching 
purposes consider to use 
[ganskef/LittleProxy-parent](https://github.com/ganskef/LittleProxy-parent). 
See [Aldo Cortesi](http://corte.si/posts/code/mitmproxy/howitworks/index.html) 
for a detailed description of proxy interception processes. 

### Get it up and running

Execute this command: `java -jar littleproxy-mitm-1.1.0-beta2b-shade.jar` Java is 
required to be installed on your System. 

The first run creates the key store for your Certificate Authority. It's used to 
generate server certificates on the fly. The ```littleproxy-mitm.pem``` file 
have to be imported in your browser or within the systems certificates, Mozilla 
for example:

<img src="https://github.com/ganskef/LittleProxy-mitm/blob/master/import-mozilla-1.png" height="250">
<img src="https://github.com/ganskef/LittleProxy-mitm/blob/master/import-mozilla-2.png" height="250">

You have to set your browsers proxy settings to 9090. It's hard coded in the 
simple Launcher class. You may chose an other implementation, of course.

### Important Security Note

**Please use your browser directly for every security-critical transmission.** 
Mozilla Firefox and Google Chrome implements her own certificate handling for a 
reason. Handling security in Java like here must be less secure in most 
situations. See http://www.cs.utexas.edu/~shmat/shmat_ccs12.pdf "The Most 
Dangerous Code in the World: Validating SSL Certificates in Non-Browser 
Software".

### Getting the library

Add this dependency to your Maven build:

```xml
<dependency>
   <groupId>com.github.ganskef</groupId>
   <artifactId>littleproxy-mitm</artifactId>
   <version>1.1.0-beta2b</version>
</dependency>
```
The version corresponds to LittleProxy since the intention was to integrate it 
as a module. **Note:** the 1.1.0-beta2 version points to the wrong LittleProxy 
version 1.1.0-beta2. Please use 1.1.0-beta2b with an additional b therefore.

### Wiring everything together

Once you've included LittleProxy-mitm, you can start the server with the following:

```java
HttpProxyServer server =
    DefaultHttpProxyServer.bootstrap()
        .withPort(9090) // for both HTTP and HTTPS
        .withManInTheMiddle(new CertificateSniffingMitmManager())
        .start();
```

Please give an `Authority` in the constructor to personalize your application. 
You impersonate certificates which is normally a bad thing. You have to describe 
the reason for.

Please refer to the documentation of 
[LittleProxy](https://github.com/adamfisk/LittleProxy) and the Javadoc of 
`org.littleshoot.proxy.HttpFilters` to filter HTTP/S contents.

### Resolving URI in case of HTTPS

Mostly you will need an URL to handle content in your filters. With HTTP it's 
provided by `originalRequest.getUri()`, but with HTTPS you have to get the host 
name from the initiating `CONNECT` request. Therefore you have to do something 
like this in your `FiltersSource` implementation: 

```java
    private static final AttributeKey<String> CONNECTED_URL = AttributeKey.valueOf("connected_url");

    @Override
    public HttpFilters filterRequest(HttpRequest originalRequest, ChannelHandlerContext clientCtx) {
        String uri = originalRequest.getUri();
        if (originalRequest.getMethod() == HttpMethod.CONNECT) {
            if (clientCtx != null) {
                String prefix = "https://" + uri.replaceFirst(":443", "");
                clientCtx.channel().attr(CONNECTED_URL).set(prefix);
            }
            return new HttpFiltersAdapter(originalRequest, clientCtx);
        }
        String connectedUrl = clientCtx.channel().attr(CONNECTED_URL).get();
        if (connectedUrl == null) {
            return new MyHttpFilters(uri);
        }
        return new MyHttpFilters(connectedUrl + uri);
    }
```

 * On `CONNECT` you must **always** return a `HttpFiltersAdapter`, since it has 
 to  bypass all filtering. 
 * Without a saved `connected_url` in the context it's plain HTTP, no HTTPS.
 * Following requests on this channel have to concatenate the saved 
 `connected_url` with the URI from the `originalRequest`.

### Workarounds for Known Problems

 * HTTPS fails with Exception: Handshake has already been started on Android Version 5+ (https://github.com/netty/netty/issues/4718). It's fixed with [PR #4767](https://github.com/netty/netty/pull/4764). Using Netty 4.1.0.CR2-SNAPSHOT MITM works well with Android 5.0, 5.1, and 6.0, just as Java platforms too.

 * Connection failure with some HTTPS sites like https://www.archlinux.org/ for example. You have to use [Java Cryptography Extension](http://en.wikipedia.org/wiki/Java_Cryptography_Extension) to fix it.
```
387481 2015-05-19 21:34:39,061 WARN  [LittleProxy-ProxyToServerWorker-6] impl.ProxyToServerConnection - (HANDSHAKING) [id: 0x7e0de7f2, /192.168.178.30:1475 => www.archlinux.org/66.211.214.131:443]: Caught exception on proxy -> web connection
io.netty.handler.codec.DecoderException: java.lang.RuntimeException: Could not generate DH keypair
    at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:346)
...
Caused by: java.security.InvalidAlgorithmParameterException: Prime size must be multiple of 64, and can only range from 512 to 1024 (inclusive)
    at com.sun.crypto.provider.DHKeyPairGenerator.initialize(DHKeyPairGenerator.java:120)
...
```
 * I'm not a natural English speaker/writer. So feel free to fix me if I'm wrong 
 (or always in generally) and don't feel sad about a phrase.

----
