User Input → Is it a meta-command? (.exit, .tables, etc.)
          ↓
          No? → Compile it (prepare_statement)
          ↓
          Execute it (execute_statement)

----------------------------------------------------
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

What's happening? This creates a type that can only have two values: success or unrecognized.

Why? Instead of just returning 0 or 1 (confusing!), you return META_COMMAND_SUCCESS (clear!). It's like having a traffic light with named colors instead of just numbers.
----------------------------------------------------
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
   StatementType type;
} Statement;

What's happening? You're creating a container that holds what kind of SQL command the user wants to run.
Why? Right now it's just the type (insert or select), but later you'll add more info like "insert WHAT?" This is your "bytecode" - the internal representation of the command.
Analogy: It's like a food order ticket:

Type: "Pizza" or "Burger"
(Later you'll add: toppings, size, etc.)
---------------------------------------------------
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    close_input_buffer(input_buffer);
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}

What's happening? This handles commands that start with . (these are NOT SQL, they're special database commands)
Why separate this? Because .exit, .tables, .help aren't SQL - they're control commands for your database tool itself.
-----------------------------------------------------
