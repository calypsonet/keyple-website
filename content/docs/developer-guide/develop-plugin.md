---
title: Develop a Plugin
type: book
toc: true
draft: false
weight: 330
---

## Overview
In order to provide an easy way to port a Keyple applications from a device with a specific card reader to another, a plugin system
as been developed. When a developer wants to include Keyple features within his project, he has to initialize the SmartCardService by providing a
plugin Factory available with in the plugin components. The plugin to use depends on the target device and running environment.

For example, for an android device with native NFC we'll use KeypleAndroidNfcPlugin.

```gradle
implementation 'org.eclipse.keyple:keyple-android-plugin-nfc:$keyple_version'
```

```kotlin
SmartCardService.getInstance().registerPlugin(AndroidNfcPluginFactory())
```

Keyple is already provided with various plug-ins ready to use. For example Android NFC or PC/SC reader plugins.

But users of the Keyple API may need to use it on new hardware not covered with existing plugins. In this case, a new 
one must be developed. 

The purpose of this guide is to accompany developers in this process.   

## Class diagram of plugin package

{{< figure library="true" src="plugin-development/class/Plugin_Class_Full.png" title="" >}} 

## Steps
Plugin development relies on 3 main steps, each one consinsiting of implementing a set of abstract classes and methods of
plugin package from Keyple Core API:
1. Import Keyple dependency
1. Implement a Keyple Reader
1. Implement a Keyple Plugin
1. Implement a Keyple Plugin Factory.


## Imports 
Your plugin will use be based upon Keyple Core libraries:

```gradle
implementation "org.eclipse.keyple:keyple-java-core:$keyple_version"
```

## Implement Keyple Reader
The first step of a Keyple plugin development is the implementation of Keyple Reader Interface (org.eclipse.keyple.core.Reader). 
This implementation with use hardware's native smartcard sdk to map interface used by Keyple API. 

This implementation of a local reader must be done through the extension of two abstract class provided within the sdk whose the choice depends
on expected behaviour of the reader:
* **AbstractLocalReader**: Basic abstract class to use for local reader implementation.
* **AbstractObservableLocalReader**: This abstract class is used to manage the matter of observing card events in the case of a local reader 
(ie: card insertion, card removal..).

{{< figure library="true" src="plugin-development/class/readers.png" title="Abstract classes to extends for Local Reader implementation" >}} 

Once chosen, the Abstract class must be extended by the new reader class and abstract methods must be implemented. Please refer to your native reader
documentation to implement this elements.

### Implementation of AbstractLocalReader's abstract classes 

Relying on  native libraries capacities of the device, implementations to be done are:

* boolean checkCardPresence()
* byte[] getATR()
* openPhysicalChannel()
* closePhysicalChannel()
* boolean isPhysicalChannelOpen()
* boolean isCurrentProtocol(String readerProtocolName)
* byte[] transmitApdu(byte[] apduIn)
* activateReaderProtocol(String readerProtocolName)
* deactivateReaderProtocol(String readerProtocolName)
* isContactless()

### Implementation of AbstractObservableLocalReader's abstract classes 

#### onEvent()
Will all to handle events like card insertion, card removal...

### ObservableReaderNotifiers
Observable reader's notification is set up by implementing interfaces. Developer has to choose how the reader should behave regarding
native abilities of the device. It may involve a few more methods to implement.

Interfaces:
| Card Insertion                  | Card Removal                       
|---------------------------------|------------------------------------
| WaitForCardInsertionAutonomous  | WaitForCardRemovalAutonomous |
| WaitForCardInsertionBlocking    | WaitForCardRemovalBlocking|
| WaitForCardInsertionNonBlocking | WaitForCardRemovalNonBlocking|
| ----                            | WaitForCardRemovalDuringProcessing|


Features:
|Type| Description                                                                                                                                                                         |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Autonomous**     | Interface to be implemented by readers that have a fully integrated management of card  communications for card insertion/removal detection                                         |
| **Blocking**      | Interface to be implemented by readers that are autonomous in the management of waiting for the  insertion/removal of a card and that provide a method to wait for it indefinitely. |
| **Non Blocking**   | Interface to be implemented by readers that require an active process to detect the card insertion /removal.                                                                        |
| **During Process** |  Interface to be implemented by readers able to detect a card __removal__ during processing,  between two APDU commands.                                                                 |

### Examples of implementation
#### checkCardPresence()
Allow Keyple to check if the secure elements is present within the reader (inserted, in NFC field...)

OMAPI Example
```kotlin
//nativeReader is an instance of android.se.omapi.Reader. It natively provides a method to check Card presence.
override fun checkCardPresence(): Boolean {
    return nativeReader.isSecureElementPresent
}
```

Android NFC Example
```kotlin
//When a SE is presented in the NFC field, we can get a tagproxy object. When the SE is removed, 
//this value is reinited. So when it is not null, we can assume the SE is currently in the field.
//The presence information is not directly available be we can find a way to implement checkCardPresence()
public override fun checkCardPresence(): Boolean {
    return tagProxy != null
}
```

PC/SC Example
```java
public override fun checkCardPresence(): Boolean {
    //Terminal is an instance of CardTerminal provided by javax.smartcardio
    return terminal.isCardPresent();
}
```

#### getAtr()
OMAPI Exemple
```kotlin
override fun getATR(): ByteArray? {
    //Session is a native object of android.se.omapi package
    return session?.let {
        it.atr
    }
}
```

Android NFC Example
```kotlin
public override fun getATR(): ByteArray? {
    //TagProxy is an object mapping android.nfc.tech.TagTechnology. Atr is obtained from data of this object (depending of protocol)
    val atr = tagProxy?.atr
    return if (atr?.isNotEmpty() == true) atr else null
}
```

PC/SC Example
```java
public override fun checkCardPresence(): Boolean {
    //Card is an instance of card provided by javax.smartcardio
    return return card.getATR().getBytes();;
}
```
#### openPhysicalChannel()
OMAPI Exemple
```kotlin
@Throws(KeypleReaderIOException::class)
override fun openPhysicalChannel() {
    try {
        //nativeReader is an instance of import android.se.omapi.Reader
        session = nativeReader.openSession()
    } catch (e: IOException) {
        throw KeypleReaderIOException("IOException while opening physical channel.", e)
    }
}
```

Android NFC Example
```kotlin
@Throws(KeypleReaderIOException::class)
public override fun openPhysicalChannel() {
    //TagProxy is a wrapper we created in the plugin
    //To handle android.nfc.Tag
    if (tagProxy?.isConnected != true) {
        try {
            tagProxy?.connect()
        } catch (e: IOException) {
            throw KeypleReaderIOException("Error while opening physical channel", e)
        }
    }
}
```

PC/SC Example
```java
public override fun checkCardPresence(): Boolean {
    //card is an instance of Card provided by javax.smartcardio
    if (card == null) {
        this.card = this.terminal.connect(parameterCardProtocol);
        } else {
          logger.debug("[{}] Opening of a card physical channel in shared mode.", this.getName());
        }
      }
    //CardChannel is an instance of card provided by javax.smartcardio
    this.channel = card.getBasicChannel();
}
```

#### closePhysicalChannel()
OMAPI Exemple
```kotlin
override fun closePhysicalChannel() {
    //openChannel is an instance of android.se.omapi.Channel
    openChannel?.let {
        it.session.close()
        openChannel = null
    }
}
```

Android NFC Example
```kotlin
@Throws(KeypleReaderIOException::class)
public override fun closePhysicalChannel() {
    try {
        //TagProxy is a wrapper we created in the plugin
        //To handle android.nfc.Tag
        tagProxy?.close()
    } catch (e: IOException) {
        throw KeypleReaderIOException("Error while closing physical channel", e)
    } finally {
        tagProxy = null
    }
}
```

PC/SC Example
```java
protected void closePhysicalChannel() {
    try {
     //card is an instance of Card provided by javax.smartcardio
      if (card != null) {
        channel = null;
        card.disconnect(true);
        card = null;
      }
    } catch (CardException e) {
      throw new KeypleReaderIOException("Error while closing physical channel", e);
    }
}
```

#### isPhysicalChannelOpen()
OMAPI Exemple
```kotlin
    override fun isPhysicalChannelOpen(): Boolean {
        //Session is an instance of android.se.omapi.Session
        return session?.isClosed == false
    }
```

Android NFC Example
```kotlin
    public override fun isPhysicalChannelOpen(): Boolean {
        //TagProxy is a wrapper we created in the plugin
        //To handle android.nfc.Tag
        return tagProxy?.isConnected == true
    }
```

PC/SC Example
```java
protected void closePhysicalChannel() {
    try {
     //card is an instance of Card provided by javax.smartcardio
      if (card != null) {
        channel = null;
        card.disconnect(true);
        card = null;
      }
    } catch (CardException e) {
      throw new KeypleReaderIOException("Error while closing physical channel", e);
    }
}
```

#### transmitApdu(byte[] apduIn)
OMAPI Exemple
```kotlin
    override fun transmitApdu(apduIn: ByteArray): ByteArray {
        // Initialization
        Timber.d("Data Length to be sent to tag : " + apduIn.size)
        Timber.d("Data in : " + ByteArrayUtil.toHex(apduIn))
        var dataOut = byteArrayOf(0)
        try {
            openChannel.let {
                dataOut = it?.transmit(apduIn) ?: throw IOException("Channel is not open")
            }
        } catch (e: IOException) {
            throw KeypleReaderIOException("Error while transmitting APDU", e)
        }

        Timber.d("Data out : " + ByteArrayUtil.toHex(dataOut))
        return dataOut
    }
```

Android NFC Example
```kotlin
    public override fun transmitApdu(apduIn: ByteArray): ByteArray {
        Timber.d("Send data to card : ${apduIn.size} bytes")
        return with(tagProxy) {
            if (this == null) {
                throw KeypleReaderIOException(
                        "Error while transmitting APDU, invalid out data buffer")
            } else {
                try {
                    val bytes = transceive(apduIn)
                    if (bytes.size <2) {
                        throw KeypleReaderIOException(
                                "Error while transmitting APDU, invalid out data buffer")
                    } else {
                        Timber.d("Receive data from card : ${ByteArrayUtil.toHex(bytes)}")
                        bytes
                    }
                } catch (e: IOException) {
                    throw KeypleReaderIOException(
                            "Error while transmitting APDU, invalid out data buffer", e)
                } catch (e: NoSuchElementException) {
                    throw KeypleReaderIOException("Error while transmitting APDU, no such Element", e)
                }
            }
        }
    }
```

PC/SC Example
```java
protected byte[] transmitApdu(byte[] apduIn) {
    ResponseAPDU apduResponseData;
    
    if (channel != null) {
      try {
        apduResponseData = channel.transmit(new CommandAPDU(apduIn));
      } catch (CardException e) {
        throw new KeypleReaderIOException(this.getName() + ":" + e.getMessage());
      } catch (IllegalArgumentException e) {
        // card could have been removed prematurely
        throw new KeypleReaderIOException(this.getName() + ":" + e.getMessage());
      }
    } else {
      // could occur if the card was removed
      throw new KeypleReaderIOException(this.getName() + ": null channel.");
    }

return apduResponseData.getBytes();
}
```

#### activateReaderProtocol(String readerProtocolName)
OMAPI Exemple
```kotlin
    override fun activateReaderProtocol(readerProtocolName: String?) {
        // do nothing as protocol ils not relevant for contact PO
    }
```

Android NFC Example
```kotlin
    override fun activateReaderProtocol(readerProtocolName: String?) {
    
        if (!protocolsMap.containsKey(readerProtocolName)) {
            //we map the requested protocol to a custom android protocol
            protocolsMap.put(readerProtocolName!!, AndroidNfcProtocolSettings.getSetting(readerProtocolName!!)!!)
        }
    }
```

PC/SC Example
```java
  protected void activateReaderProtocol(String readerProtocolName) {
    if (!PcscProtocolSetting.getSettings().containsKey(readerProtocolName)) {
      throw new KeypleReaderProtocolNotSupportedException(readerProtocolName);
    }
  }
```

#### deactivateReaderProtocol(String readerProtocolName)
OMAPI Exemple
```kotlin
    override fun deactivateReaderProtocol(readerProtocolName: String?) {
        // do nothing
    }
```

Android NFC Example
```kotlin
    override fun deactivateReaderProtocol(readerProtocolName: String?) {
        if (protocolsMap.containsKey(readerProtocolName)) {
            protocolsMap.remove(readerProtocolName)
        }
        Timber.d("${getName()}: Deactivate protocol $readerProtocolName.")
    }
```

PC/SC Example
```java
  protected void deactivateReaderProtocol(String readerProtocolName) {
    if (!PcscProtocolSetting.getSettings().containsKey(readerProtocolName)) {
      throw new KeypleReaderProtocolNotSupportedException(readerProtocolName);
    }

    if (logger.isDebugEnabled()) {
      logger.debug("{}: Deactivate protocol {}.", getName(), readerProtocolName);
    }
  }
```


#### isCurrentProtocol(String readerProtocolName)
OMAPI Exemple
```kotlin
    override fun isCurrentProtocol(readerProtocolName: String?): Boolean {
        return AndroidOmapiSupportedProtocols.ISO_7816_3.name == readerProtocolName
    }
```

Android NFC Example
```kotlin
    override fun isCurrentProtocol(readerProtocolName: String?): Boolean {
        return readerProtocolName == null || protocolsMap.containsKey(readerProtocolName) && protocolsMap[readerProtocolName] == tagProxy?.tech
    }
```

PC/SC Example
```java
  protected boolean isCurrentProtocol(String readerProtocolName) {
    String protocolRule = PcscProtocolSetting.getSettings().get(readerProtocolName);
    String atr = ByteArrayUtil.toHex(card.getATR().getBytes());
    return Pattern.compile(protocolRule).matcher(atr).matches();
  }
```

#### isContactless()
OMAPI Exemple
```kotlin
    override fun isContactless(): Boolean {
        //OMAPI is a contact Reader
        return false
    }
```

Android NFC Example
```kotlin
    override fun isContactless(): Boolean {
        //NFC is a contactless Reader
        return true
    }
```

PC/SC Example
```java
  public boolean isContactless() {
    //isContactless is a custom variable that can be set by the plugin 
    //as PC/SC can be a contactless or contact reade.
    if (isContactless == null) {
      /* First time initialisation, the transmission mode has not yet been determined or fixed explicitly, let's ask the plugin to determine it (only once) */
      isContactless =
          ((AbstractPcscPlugin) SmartCardService.getInstance().getPlugin(getPluginName()))
              .isContactless(getName());
    }
    return isContactless;
  }
```

