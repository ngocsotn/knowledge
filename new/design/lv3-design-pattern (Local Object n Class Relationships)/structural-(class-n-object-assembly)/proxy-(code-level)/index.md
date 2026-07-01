# Proxy Pattern (Code Level)

Proxy provides a surrogate or placeholder object to control access to another target object.
- **Types:** Virtual Proxy (lazy loading), Protective Proxy (access control), Logging/Auditing Proxy.

## Interview Questions & Answers

### Q1: How do you implement lazy loading of a heavy object using the Virtual Proxy pattern?
- **Answer:** The Proxy class implements the identical interface as the heavy target object. However, the proxy does not instantiate the heavy object upon creation. It only instantiates the target inside its action methods upon the first actual method invocation, caching the instance for future calls, saving memory until the object is actually used.
