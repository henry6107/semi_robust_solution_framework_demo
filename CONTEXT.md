# Robust Solution Framework

This context describes how machine behavior is organized and how exclusive control of physical capabilities is coordinated.

## Language

**BaseUnit**:
A reusable machine capability that owns direct control of one physical resource.
_Avoid_: Device wrapper, hardware helper

**Service**:
A PackML-managed operation that requests and uses one or more BaseUnits to perform machine behavior.
_Avoid_: Command handler, operation block

**Resource**:
The exclusive right to control a BaseUnit for the duration of one Service operation.
_Avoid_: Lock, mutex

**Owner**:
The Service instance that currently holds a Resource.
_Avoid_: Caller, user

**Acquire**:
The act by which a Service becomes the Owner of a free Resource.
_Avoid_: Lock, reserve

**Release**:
The act by which the Owner makes a Resource available to another Service.
_Avoid_: Unlock, free
