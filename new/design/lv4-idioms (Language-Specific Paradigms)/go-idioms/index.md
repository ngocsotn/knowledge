# Go Idioms

## Topics

- Explicit Error Returning: Avoiding try-catch exceptions entirely. Instead, functions return data and an error object as a tuple (value, err := doSomething()), forcing explicit error checks.
- Empty Interface (interface{} / any): Using an interface with no methods to allow a variable to hold absolutely any data type.
