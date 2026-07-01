# Visitor Pattern

Visitor lets you define a new operation to be executed on the elements of an object structure without changing the classes of the elements on which it operates.
- **Mechanism:** Double dispatch. Element classes implement `accept(Visitor v)`, which calls `v.visit(this)` back to execute Visitor logic.

## Interview Questions & Answers

### Q1: Why is the Visitor pattern highly utilized in compiler AST structures?
- **Answer:** Separation of concerns. An Abstract Syntax Tree (AST) contains diverse element classes (e.g., `AssignmentNode`, `IfNode`). To execute distinct operations (e.g., type checking, code generation, pretty printing) without polluting node classes, you write separate visitors like `TypeCheckVisitor` and `CodeGeneratorVisitor`, keeping nodes static.
