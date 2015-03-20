## Introduction ##

pybonjour provides a pure-Python interface to [Apple Bonjour](http://developer.apple.com/networking/bonjour/) and compatible [DNS-SD](http://www.dns-sd.org/) libraries (such as [Avahi](http://avahi.org/)).  It allows [Python](http://python.org/) scripts to take advantage of [Zero Configuration Networking](http://www.zeroconf.org/) (Zeroconf) to register, discover, and resolve services on both local and wide-area networks.  Since pybonjour is implemented in pure Python, scripts that use it can easily be ported to Mac OS X, Windows, Linux, and other systems that run Bonjour.

## Requirements ##

pybonjour requires Python 2.4 or later.  It also depends on [ctypes](http://docs.python.org/library/ctypes.html) (version 1.0.1 or later), which is part of the standard library in Python 2.5 and later but must be [downloaded](http://python.net/crew/theller/ctypes/#downloads) and installed separately for Python 2.4.

No additional software is required to use pybonjour under Mac OS X.  To use it under Windows, the [Bonjour for Windows](http://developer.apple.com/networking/bonjour/download/) package must be installed on your system.  Most Linux systems use Avahi rather than Bonjour for DNS-SD.  If this is the case on your system, you'll need to install Avahi's Bonjour compatibility library.  (Under [Ubuntu](http://www.ubuntu.com/), the package to install is [libavahi-compat-libdnssd1](http://packages.ubuntu.com/jaunty/libavahi-compat-libdnssd1).)  Otherwise, or to use pybonjour under other POSIX systems, you must [download](http://developer.apple.com/networking/bonjour/download/), compile, and install Bonjour from source.

## Examples ##

The following scripts are included in the `examples` directory of the pybonjour source distribution.

### Registering a Service ###

#### register.py Script ####

```
import select
import sys
import pybonjour


name    = sys.argv[1]
regtype = sys.argv[2]
port    = int(sys.argv[3])


def register_callback(sdRef, flags, errorCode, name, regtype, domain):
    if errorCode == pybonjour.kDNSServiceErr_NoError:
        print 'Registered service:'
        print '  name    =', name
        print '  regtype =', regtype
        print '  domain  =', domain


sdRef = pybonjour.DNSServiceRegister(name = name,
                                     regtype = regtype,
                                     port = port,
                                     callBack = register_callback)

try:
    try:
        while True:
            ready = select.select([sdRef], [], [])
            if sdRef in ready[0]:
                pybonjour.DNSServiceProcessResult(sdRef)
    except KeyboardInterrupt:
        pass
finally:
    sdRef.close()
```

#### Example Output ####

```
$ python register.py TestService _test._tcp 1234
Registered service:
  name    = TestService
  regtype = _test._tcp.
  domain  = local.
```

### Browsing for and Resolving Services ###

#### browse\_and\_resolve.py Script ####

```
import select
import sys
import pybonjour


regtype  = sys.argv[1]
timeout  = 5
resolved = []


def resolve_callback(sdRef, flags, interfaceIndex, errorCode, fullname,
                     hosttarget, port, txtRecord):
    if errorCode == pybonjour.kDNSServiceErr_NoError:
        print 'Resolved service:'
        print '  fullname   =', fullname
        print '  hosttarget =', hosttarget
        print '  port       =', port
        resolved.append(True)


def browse_callback(sdRef, flags, interfaceIndex, errorCode, serviceName,
                    regtype, replyDomain):
    if errorCode != pybonjour.kDNSServiceErr_NoError:
        return

    if not (flags & pybonjour.kDNSServiceFlagsAdd):
        print 'Service removed'
        return

    print 'Service added; resolving'

    resolve_sdRef = pybonjour.DNSServiceResolve(0,
                                                interfaceIndex,
                                                serviceName,
                                                regtype,
                                                replyDomain,
                                                resolve_callback)

    try:
        while not resolved:
            ready = select.select([resolve_sdRef], [], [], timeout)
            if resolve_sdRef not in ready[0]:
                print 'Resolve timed out'
                break
            pybonjour.DNSServiceProcessResult(resolve_sdRef)
        else:
            resolved.pop()
    finally:
        resolve_sdRef.close()


browse_sdRef = pybonjour.DNSServiceBrowse(regtype = regtype,
                                          callBack = browse_callback)

try:
    try:
        while True:
            ready = select.select([browse_sdRef], [], [])
            if browse_sdRef in ready[0]:
                pybonjour.DNSServiceProcessResult(browse_sdRef)
    except KeyboardInterrupt:
        pass
finally:
    browse_sdRef.close()
```

#### Example Output ####

```
$ python browse_and_resolve.py _test._tcp
Service added; resolving
Resolved service:
  fullname   = TestService._test._tcp.local.
  hosttarget = bumble.local.
  port       = 1234
...
```