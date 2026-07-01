# Command Pattern (Code Level)

Command encapsulates a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.
- **Core Role:** Encapsulates the receiver, the action, and parameter arguments into a single object implementing `execute()` and `undo()`.

## Interview Questions & Answers

### Q1: How do you implement a multi-level Undo history using the Command pattern?
- **Answer:** Maintain an array or stack of executed Command objects. When a user executes an action, the command is executed and pushed onto the history stack. When "Undo" is clicked, the application pops the top command from the stack and calls its `undo()` method, reversing the state change. You can maintain a separate "Redo" stack for reversing undoes.
