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
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }
  return PREPARE_UNRECOGNIZED_STATEMENT;
}

What's happening? This is your "SQL Compiler" - it reads the text and figures out what the user wants.
Note the difference:

strncmp(..., "insert", 6) - checks only first 6 characters because insert will have data after it: insert 1 john john@email.com
strcmp(..., "select") - checks the whole thing because select (for now) is just one word

It fills in the statement object with the type, then returns success or failure.
----------------------------------------------------
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}

What's happening? This is your "Virtual Machine" - it actually DOES what the statement says (or will, once you write that code).
Right now it's just stubs (placeholders) because you haven't built the actual database yet. In Part 3, you'll replace these printf statements with real insert/select logic.

----------------------------------------------------
