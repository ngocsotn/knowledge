# Mediator Pattern

Mediator defines an object that encapsulates how a set of objects interact, promoting loose coupling by keeping objects from referring to each other explicitly.
- **Usage:** Chatroom mediator connecting multiple users; complex UI dashboards coordinate sibling widgets through a single parent controller.

## Interview Questions & Answers

### Q1: What is the main trade-off of the Mediator pattern?
- **Answer:** God Object vulnerability. While the Mediator pattern successfully decouples multiple sibling classes from talking directly to each other, all the complex communication and coordination logic becomes centralized inside the Mediator class. Over time, the mediator can grow into an unmaintainable, tightly-coupled God Object.
