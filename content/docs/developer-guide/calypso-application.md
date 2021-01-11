---
title: Calypso application
type: book
toc: true
draft: false
weight: 330
---

## Overview

Keyple API was designed to support an extension system. On top of **Keyple Core**, APIs can be developed to extend
Keyple features. For example, Calypso Network Association provides **Keyple Calypso Extension**. 

The use of **Keyple Calypso Extension** open the ability to operate commands with a calypso Portable Object and to manage a 
secure calypso transaction.

The diagram below shows the role of the **Keyple Calypso Extension** components in the software layers for a standalone application :

{{< figure library="true" src="calypso-app-development/component/Local_Component_Overview.svg" title="" >}}

## Before you start
1. In pre-requisite, have knowledge of the standard Calypso.
1. Read the [common concepts]({{< relref "common-concepts.md" >}}) page and become familiar with the basic concepts on which **Keyple** is based.
1. Any implementation of a Keyple application starts with the implementation of **Keyple Core**, please study the [workflow]({{< relref "standalone-application.md" >}}) proposed in Standalone application guide.
1. Read chapter [Concepts](#concepts) to understand the main terms and concepts of the  **Keyple Calypso Extension** API.
1. Explore the [Keyple Calypso API](#keyplecalypsoapi) to discover all the possibilities offered by **Keyple Calypso Extension**.
1. Take inspiration from the [examples](#examples).
1. Follow the explanations given in the [Build your first app]({{< relref "build-your-first-app" >}}) section to configure your environment.
1. Using the [Java components]({{< ref "components-java/core/_index.md" >}}) or [C++ components]({{< ref "components-cpp/core/_index.md" >}}) pages, import **Keyple Core** into your project.
1. Using the [Java components]({{< ref "components-java/extensions/calypso.md" >}}) or [C++ components]({{< ref "components-cpp/extensions/calypso.md" >}}) pages, import **Keyple Calypso Extension** into your project.
1. Start playing with **Keyple**.

## Concepts
Here are the main concepts to keep in mind before continuing to read this developer guide :

### Calypso PO
Concentrate all known informations about the Personal Object being processed. Accessible informations are
* The application identification fields (revision/version, class, DF name, serial number, ATR, issuer)
* The indication of the presence of optional features (Stored Value, PIN, Rev3.2 mode, ratification management)
* The management information of the modification buffer
* The invalidation status
* The files, counters, SV data read or modified during the execution of the processes defined  by PoTransaction

Calypso PO fields are populated from a CardSelectionResponse obtained through the process of a PO selection.

```java
    ...
    //Retrieve PO's informations
    String atr = calypsoPo.getAtr()
    byte[] applicationSerial+Number = calypsoPo.getApplicationSerialNumber()
    //SFI_EventLog = 0x08
    ElementaryFile efEventLog = calypsoPo.getFileBySfi(SFI_EventLog);
    //SFI_EnvironmentAndHolder = 0x07
    ElementaryFile efEnvironmentAndHolder =calypsoPo.getFileBySfi(SFI_EnvironmentAndHolder);
    ...
```

### ElementaryFile
Object containing the description of a Calypso Elementary File. Can be retrieved from Calypso PO using its SFI. 

```java
    ...
    ElementaryFile efEventLog = calypsoPo.getFileBySfi(SFI_EventLog);
    //Read data content of sevent log elemenatary file
    String eventLog = ByteArrayUtil.toHex(efEventLog.getData().getContent());
    ...
```

### Calypso SAM
Concentrates all the informations we know about the SAM currently selected. Accessible informations are:
 * The Sam Revision
 * The Serial number
 * The Platform identifier
 * The Application Type
 * The Application SubType
 * The Software Issuer identifier
 * The Software Version number
 * The Software Revision number

Calypso SAM fields are populated by analysis of the ATR within a CardSelectionResponse obtained through the process of a SAM selection.

```java
    ...
    byte[] serialNumber = calypsoSam.getSerialNumber()
    ...
```

### PoSecuritySettings
Concentrate the security settings involved in Calypso Secure Sessions:
* A reference to the Sam resource
* The default KIF  
* The default KVC
* The default Key Record Number
* The modification mode
* The ratification mode
* The pin transmission mode
* The default Pin Ciphering Key
* The SV Get Log Reade mode
* the SV Negative balance

 The fields are populated with default values when the object is instantiated but can be customized to adjust the settings
 to the application needs.
 
 ```java
     ...
     poSecuritySettings.getSessionModificationMode()
     ...
 ```

Secure session settings can be finely tuned with PoSecuritySettings and a set of enums provided by Keyple:

#### Modification mode
Indicates whether the secure session can be closed and reopened to manage the limitation of the PO buffer memory.
| ModificationMode| Description                       
|---------------------------------|------------------------------------
|**`ATOMIC`**|The secure session is atomic.
|**`MULTIPLE`**|Several secure sessions can be chained (to manage the writing of large amounts of data).

 ```java
     ...
    PoSecuritySettings poSecuritySettings =
        new PoSecuritySettings.PoSecuritySettingsBuilder(samResource)
            .svGetLogReadMode(SvSettings.LogRead.ALL)
            .build();
     ...
 ```

#### Ratification mode
The ratification mode defines the behavior of processClosing regarding the ratification process.
| RatificationMode| Description                       
|---------------------------------|------------------------------------
|**`CLOSE_RATIFIED`**|Close session with ratification.
|**`CLOSE_NOT_RATIFIED`**|Close session without ratification

 ```java
     ...
    PoSecuritySettings poSecuritySettings =
        new PoSecuritySettings.PoSecuritySettingsBuilder(samResource)
            .ratificationMode(RatificationMode.CLOSE_RATIFIED)
            .build();
     ...
 ```

#### Pin Transmission Mode
Defines the PIN transmission modes: plain or encrypted.
| PinTransmissionMode|                      
|---------------------------------|
|**`PLAIN`**|
|**`ENCRYPTED`**|

## Basic Operations

### PoSelection

Service extending Keyple Core Abstract Card Selection to manage specific features of Calypso POs during the selection:
* Send APDU Commands to the POs right after the card selection.
* Produce a Calypso PO from the CardSelectionResponse. The object is filled with the PO identification data from the FCI and the
possible responses to additional APDU commands executed after the selection.

```java
    ...
    // Prepare a Calypso PO selection
    CardSelectionsService cardSelectionsService = new CardSelectionsService();

    // Setting of an AID based selection of a Calypso REV3 PO
    // Select the first application matching the selection AID whatever the card communication
    // protocol keep the logical channel open after the selection

    // Calypso selection: configures a PoSelection with all the desired attributes to
    // make the selection and read additional information afterwards
    PoSelection poSelection =
        new PoSelection(
            PoSelector.builder()
                .cardProtocol(ContactlessCardCommonProtocols.ISO_14443_4.name())
                .aidSelector(
                    CardSelector.AidSelector.builder().aidToSelect(CalypsoClassicInfo.AID).build())
                .invalidatedPo(PoSelector.InvalidatedPo.REJECT)
                .build());

    // Prepare the reading order.
    poSelection.prepareReadRecordFile(
        CalypsoClassicInfo.SFI_EnvironmentAndHolder, CalypsoClassicInfo.RECORD_NUMBER_1);

    // Add the selection case to the current selection (we could have added other cases
    // here)
    cardSelectionsService.prepareSelection(poSelection);
    CalypsoPo calypsoPo = (CalypsoPo) cardSelectionsService.processExplicitSelections(poReader).getActiveSmartCard();
    ...
```

## SamSelection
Service extending Keyple Core Abstract Card Selection specialized to manage the specific characteristics of Calypso SAMs. 
The service provides an instance of Calypso SAM and may execute the unlock command during the selection process.

```java
    ...
    CardSelectionsService samSelection = new CardSelectionsService();

    //Sam selection parameters
    SamSelector samSelector = SamSelector.builder().samRevision(SamRevision.C1).serialNumber(".*").build();
    samSelection.prepareSelection(new SamSelection(samSelector));

    //Sam reader retrieved from registered plugin
    CardSelectionsResult cardSelectionsResult = samSelection.processExplicitSelections(samReader);
    if (!cardSelectionsResult.hasActiveSelection()) {
      throw new IllegalStateException("Unable to open a logical channel for SAM!");
    }

    //Cast selected card to CalypsoSam
    CalypsoSam calypsoSam = (CalypsoSam) cardSelectionsResult.getActiveSmartCard();
    ...
```

### PoTransaction
Service providing high-level API to manage transactions with a Calypso PO. The tied Calypso PO Object is kept and updated at
each step of using this service. 

This service workflow is composed of two steps:
* Prepare the commands to be executed to the PO
* Process the prepared commands. Regarding of commands, the presence of SAM could be mandatory.

```java
    ...
    // CardResource allow management of specific card.
    // In this example a SAM is available 
    CardResource<CalypsoPo> poResource = new CardResource<CalypsoPo>(poReader, calypsoPo);
    CardResource<CalypsoSam> samResource = new CardResource<CalypsoPo>(samReader, calypsoSam);
    
    // Configure Security settings
    // Both Reload and Debit SV logs are requested
    PoSecuritySettings poSecuritySettings =
        new PoSecuritySettings.PoSecuritySettingsBuilder(samResource)
            .svGetLogReadMode(SvSettings.LogRead.ALL)
            .build();

    PoTransaction poTransaction = new PoTransaction(poResource, poSecuritySettings);

    // Read the EventLog file at record 1 within the Session Opening
    poTransaction.prepareReadRecordFile(CalypsoClassicInfo.SFI_EventLog, CalypsoClassicInfo.RECORD_NUMBER_1);

    // Open a secure session (DEBIT level) and execute the prepared command
    poTransaction.processOpening(PoTransaction.SessionSetting.AccessLevel.SESSION_LVL_DEBIT);

    // Get and display the EventLog data
    ElementaryFile efEventLog = calypsoPo.getFileBySfi(CalypsoClassicInfo.SFI_EventLog);

    // Example of data parsing
    String eventLog = ByteArrayUtil.toHex(efEventLog.getData().getContent());

    // Prepare a SV Debit (this command could also have been placed before processOpening
    // since it is not followed by any other command)
    poTransaction.prepareSvGet(SvSettings.Operation.DEBIT, SvSettings.Action.DO);

    // Execute the prepared command
    poTransaction.processPoCommands();

    //Get updated sv balance
    int svBalance = calypsoPo.getSvBalance()

    //Get the updated SV last transaction number
    int svLastTNum = calypsoPo.getSvLastTNum()
    ...
```

#### Secure Session Access level

PoTransaction.processOpening() allows to open a Calypso Secure Session. If commands have been prepared previously, they will be executed. 

Keyple provides an enums to easily configure the Secure Session AccessLevel:

| AccessLevel| Description                       
|---------------------------------|------------------------------------
|**`SESSION_LVL_PERSO`**|Session Access Level used for personalization purposes.
|**`SESSION_LVL_LOAD`**|Session Access Level used for reloading purposes.
|**`SESSION_LVL_DEBIT`**|Session Access Level used for validating and debiting purposes. 


  
  
## Examples
Detailed use case examples can be seen here:

* [PC](https://github.com/eclipse/keyple-java/tree/master/java/example/calypso/pc)
* [Android](https://github.com/eclipse/keyple-java/tree/master/java/example/calypso/android)