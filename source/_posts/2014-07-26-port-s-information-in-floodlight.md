---
layout: post
title: Ports information in Floodlight
date: '2014-07-26 09:29'
comments: true
tags:
  - SDN
  - Openflow
  - Floodlight
  - Network
  - Java
  - SourceCode
keywords: 'SDN,Controller,Floodlight'
abbrlink: 25937
---
Question
--------
How can we get the port's information in Floodlight?

<!--more-->


Solution
--------
- The Floodlight use a `ImmutablePort` class to represent a switch port and a `IOFSwitch` class has a Portmanager which will manager all ImmutablePort.

- The content of ImmutablePort is the same as what it described in Openflow specification 1.0.
``` java
public class ImmutablePort {
    private final short portNumber;
    private final byte[] hardwareAddress;
    private final String name;
    private final EnumSet<OFPortConfig> config;
    private final boolean portStateLinkDown;
    private final OFPortState stpState;
    private final EnumSet<OFPortFeatures> currentFeatures;
    private final EnumSet<OFPortFeatures> advertisedFeatures;
    private final EnumSet<OFPortFeatures> supportedFeatures;
    private final EnumSet<OFPortFeatures> peerFeatures;
    ....
```

- The Portmanger provide some API to allow other object to fetch the ImmutablePort.

``` java
public ImmutablePort getPort(String name) {
	if (name == null) {
		throw new NullPointerException("Port name must not be null");
	}
	lock.readLock().lock();
	try {
		return portsByName.get(name.toLowerCase());
	} finally {
		lock.readLock().unlock();
	}
}

public ImmutablePort getPort(Short portNumber) {
	lock.readLock().lock();
	try {
		return portsByNumber.get(portNumber);
	} finally {
		lock.readLock().unlock();
	}
}

public List<ImmutablePort> getPorts() {
	lock.readLock().lock();
	try {
		return portList;
	} finally {
		lock.readLock().unlock();
	}
}

public List<ImmutablePort> getEnabledPorts() {
	lock.readLock().lock();
	try {
		return enabledPortList;
	} finally {
		lock.readLock().unlock();
	}
}
```

- Since the Portmanager is a private member of IOFSwitch, you can't directly use it. You must use the API provied by IOFSwitch to interact with Portmanager.

``` java
    @Override
    @JsonIgnore
    public Collection<ImmutablePort> getEnabledPorts() {
        return portManager.getEnabledPorts();
    }

    @Override
    @JsonIgnore
    public Collection<Short> getEnabledPortNumbers() {
        return portManager.getEnabledPortNumbers();
    }

    @Override
    public ImmutablePort getPort(short portNumber) {
        return portManager.getPort(portNumber);
    }

    @Override
    public ImmutablePort getPort(String portName) {
        return portManager.getPort(portName);
    }

    @Override
    @JsonProperty("ports")
    public Collection<ImmutablePort> getPorts() {
        return portManager.getPorts();
    }
```

Example
--------
Assume the type of sw is IOFSwitch.
``` java
Collection<ImmutablePort> swPorts = sw.getPorts();
Iterator<ImmutablePort> it = swPorts.iterator();
while(it.hasNext()){
    ImmutablePort port = it.next();
    //do something 
}
```
