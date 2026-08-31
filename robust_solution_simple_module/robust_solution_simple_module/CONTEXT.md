# Robust Solution Module Framework

This context defines the shared domain language for Module configuration and alarm behavior.

## Module Configuration Language

**Module Type Capacity**:
The maximum number of configured Modules belonging to one Module type.
_Avoid_: Global Module count, shared slot capacity

**Total Configured Module Capacity**:
The maximum number of Module entries in one configuration, including enabled and disabled entries; it covers every supported Module type at its Module Type Capacity.
_Avoid_: Module Type Capacity, slot count

**Slot**:
A Module's position within the storage owned by its Module type; the same slot number may identify Modules of different types.
_Avoid_: Global slot, Module ID

**Module ID**:
The non-zero, globally unique identity of an enabled Module, independent of its Module type and Slot.
_Avoid_: Slot, array index

## Alarm Lifecycle Language

**Alarm Identity**:
The stable key `(ServiceId, MainErrorId)` within one Module. Configured alarms always use `ServiceId = 0`; a published list index is never an identity.
_Avoid_: AlarmList index, Message, SourceErrorId

**Alarm Lifecycle**:
The interval from the first active occurrence of one Alarm Identity until that identity is both inactive and acknowledged and is therefore removed. A new occurrence after removal starts a new lifecycle.
_Avoid_: One PLC scan, one ErrorList snapshot

**Active Alarm**:
An alarm whose source condition currently exists. For Service alarms this means the identity was observed in the Service ErrorList during the current Module scan; for configured alarms it means the configured condition currently evaluates true.
_Avoid_: Latched Alarm, unacknowledged alarm

**Acknowledged Alarm**:
An alarm whose current lifecycle or most recent recurrence has been explicitly acknowledged by the upper controller. A recurrence clears acknowledgement.
_Avoid_: Inactive alarm, removed alarm

**Latched Alarm**:
An alarm lifecycle retained by the Module Alarm Manager and published even after becoming inactive, until the removal predicate `Active = FALSE AND Acknowledged = TRUE` is satisfied.
_Avoid_: Active-only snapshot, Service ErrorList entry

**Occurrence**:
A `FALSE -> TRUE` transition of an Alarm Identity within its current lifecycle. The first occurrence sets `OccurrenceCount = 1`; continuous active scans do not increment it.
_Avoid_: PLC cycle count, ErrorList polling count
