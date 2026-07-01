# Bridge Pattern

Bridge decouples an abstraction from its implementation so that the two can vary independently.
- **Mechanism:** Replaces multidimensional inheritance hierarchies with object composition (e.g., separating a `RemoteControl` abstraction from a `Device` implementation).

## Interview Questions & Answers

### Q1: When do you choose the Bridge pattern over standard class inheritance?
- **Answer:** Choose Bridge when you need to avoid a combinatorial explosion of subclasses. For example, if you have abstractions `TVRemote` and `RadioRemote`, and implementations `Sony` and `Samsung`, inheritance would force you to write 4 subclasses (`SonyTVRemote`, `SamsungTVRemote`, etc.). Using Bridge, you compose remote control objects with device interfaces, scaling both sides independently ($M + N$ classes instead of $M \times N$).
