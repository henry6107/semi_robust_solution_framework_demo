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
The stable key `(ServiceId, MainErrorId)` within one Module. Configured and Hook alarms always use `ServiceId = 0`; a valid configured definition owns the complete identity when it overlaps a Hook submission. Non-zero Service IDs remain independent. A published list index is never an identity.
_Avoid_: AlarmList index, Message, SourceErrorId

**Alarm Lifecycle**:
The interval from the first active occurrence of one Alarm Identity until that identity is both inactive and acknowledged and is therefore removed. A new occurrence after removal starts a new lifecycle.
_Avoid_: One PLC scan, one ErrorList snapshot

**Active Alarm**:
An alarm whose source condition currently exists. For Service alarms this means the identity was observed in the Service ErrorList during the current Module scan; for Hook alarms it means the identity was submitted from `H_UpdateAlarm()` in that scan; for configured alarms it means the latest valid configured sample evaluated true. An invalid configured sample retains the previous lifecycle state and never falls back to a Hook condition.
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

## Module Hook Language

**Hook Alarm**:
A Module-level alarm submitted by application logic after the Base Unit cyclic update, using `M_AddAlarm` within `H_UpdateAlarm`. It shares the existing Alarm Lifecycle and acknowledgement contract and has 30 reserved lifecycle slots. Service alarms have 40 slots and configured alarms have 30; the published total remains 100.

**Configured Ownership**:
A valid RuntimeConfig definition takes full precedence over the same Hook identity, including metadata, source value and alarm condition. Ownership does not depend on sample validity or whether an alarm is active. A configured takeover removes the old Hook lifecycle before acknowledgement; unrelated Hook and Service lifecycles survive configuration revision changes.

**Hook SVID**:
A descriptor submitted by `M_AddVariable` within `H_UpdateVariable` for the current Module scan. It needs no configured node registration and disappears when omitted. JSON descriptors have priority within the shared 100-entry VariableList; the descriptor ID, not its array index, is the identity.
