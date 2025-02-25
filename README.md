# Log4jHotPatch

This is a simple tool which injects a Java agent into a running JVM process. The agent will patch the `lookup()` method of all loaded `org.apache.logging.log4j.core.lookup.JndiLookup` instances to unconditionally return the string "Patched JndiLookup::lookup()". This should fix the [CVE-2021-44228](https://www.randori.com/blog/cve-2021-44228/) remote code execution vulnerability in Log4j without restarting the Java process.

This has been currently only tested with JDK 8 & 11 on Linux!

## Building

JDK 8
```
javac -XDignore.symbol.file=true -cp <java-home>/lib/tools.jar Log4jHotPatch.java
```

JDK 11
```
javac --add-exports java.base/jdk.internal.org.objectweb.asm=ALL-UNNAMED --add-exports=jdk.internal.jvmstat/sun.jvmstat.monitor=ALL-UNNAMED Log4jHotPatch.java
```

### Building a static agent

After compiling as described above, build the agent jar file as follows:
```
jar -cfm Log4jHotPatch.jar Manifest.mf *.class
```

## Running

JDK 8
```
java -cp .:<java-home>/lib/tools.jar Log4jHotPatch <java-pid>
```

JDK 11
```
java Log4jHotPatch <java-pid>
```

### Running the static agent

Simply add the agent to your java command line as follows:
```
java -classpath <class-path> -javaagent:Log4HotjPatch.jar <main-class> <arguments>
```

## Known issues

If you get an error like:
```
Exception in thread "main" com.sun.tools.attach.AttachNotSupportedException: The VM does not support the attach mechanism
	at jdk.attach/sun.tools.attach.HotSpotAttachProvider.testAttachable(HotSpotAttachProvider.java:153)
	at jdk.attach/sun.tools.attach.AttachProviderImpl.attachVirtualMachine(AttachProviderImpl.java:56)
	at jdk.attach/com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:207)
	at Log4jHotPatch.loadInstrumentationAgent(Log4jHotPatch.java:115)
	at Log4jHotPatch.main(Log4jHotPatch.java:139)
```
this means that your JVM is refusing any kind of help because it is running with `-XX:+DisableAttachMechanism`.

If you get an error like:
```
com.sun.tools.attach.AttachNotSupportedException: Unable to open socket file: target process not responding or HotSpot VM not loaded
	at sun.tools.attach.LinuxVirtualMachine.<init>(LinuxVirtualMachine.java:106)
	at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:63)
	at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:208)
	at Log4jHotPatch.loadInstrumentationAgent(Log4jHotPatch.java:182)
	at Log4jHotPatch.main(Log4jHotPatch.java:259)
```
this means you're running as a different user (including root) than the target JVM. JDK 8 can't handle patching as root user (and triggers a thread dump in the target JVM which is harmless). In JDK 11 patching a non-root process from a root process works just fine.
