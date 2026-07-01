# Memento Pattern

Memento captures and externalizes an object's internal state without violating encapsulation, allowing the object to be restored to this state later.
- **Components:** Originator (owns state), Memento (stores snapshot), Caretaker (manages history).

## Interview Questions & Answers

### Q1: Why does Memento protect encapsulation compared to simple state serialization?
- **Answer:** The Memento object is a black box to the external Caretaker class. The Caretaker cannot read or write to the memento's internal state attributes; only the Originator that created the memento has the privilege to serialize and deserialize its own state from the memento, preserving object encapsulation boundaries.
