# Module Configuration

This context describes how configured Modules are identified and placed within the machine configuration.

## Language

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
