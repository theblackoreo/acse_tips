
**Save statement before a block and retrieve the value after a block**

*VARIABLE INIZIALIZATION*

Check if an IDENTIFIER is not a scalar:
```c
t_axe_variable *v_var = getVariable(program, $2);
if(!v_var){
  yyerror("...");
  YYERROR;
}
```
If I want to inizialize a variable declare as global to use during execution in order to avoid problems it is possibile to inizialize it in the
```
program  : var_declarations { //declare it here} statement
```
 It is considered like a register.
For example
```c

int my_val;

program:
    var_declarations {
      my_val = gen_load_immediate(program, 12345);
    }
    statements EOF_TOK {....}
    ...
```

It is possible access to a token after {} even if is declared before.

example:
```c
FORALL LPAR IDENTIFIER ASSIGN exp {

           int loc = get_symbol_location(program, $3, 0);
         }
             TO exp RPAR {

             } code_block {
                int loc = get_symbol_location(program, $3, 0); //declare before $3
             }      
;
```
```c

int a = 10;

save {
    a = 8;
    write(a); // prints 8
}
write(a); //prints 10

// acse

save_statement:
         SAVE IDENTIFIER {
            // before the block copy its value into a temporary register
            temp_save = getNewRegister(program);
            int loc = get_symbol_location(program, $2, 0);
            gen_add_instruction(program, temp_save, loc, REG_0, CG_DIRECT_ALL); //add to 0 and save, unique method to save


            }code_block {
              // temp_save should be global to be accessed from here
            int loc = get_symbol_location(program, $2, 0);
            gen_add_instruction(program, temp_save, REG_0, loc, CG_DIRECT_ALL);

         }

;

```
***

**STORE GLOBAL VARIABLES**

**3 methods to store global varibales:**

1. declare as global -> not work if the statment is nested

2. Global Stack: Works when the statement is nestable

   ```c
   t_list *save_stack = NULL;
   // declare as global ^

   // to add into the stack
   save_stack = addFirst(save_stack, INTDATA(r_save)); // push

   // to pop from the stack
   int r_save = LINTDATA(save_stack);
   save_stack = removeFirst(save_stack); // pop
   ```

3. Semantic value as a variable:

  Works when the statement is nestable
  Doesn’t work when the variable must be accessible from multiple rules (not common)
  Sometime you have one or more tokens without semantic value: (for example save keyword)
  The idea is to use this token to pass variables from one semantic action to another
  STEPS:
  1. Add a type declaration to the token Acse.y %token <intval> SAVE
  2. Do NOT assign a semantic value in Acse.lex
  3. Use $n to access the variable of the token in semantic actions:
    ```c
    $1 = getNewRegister(program);
    int r_var = get_symbol_location(program, $2, 0);
    gen_add_instruction(program, $1, REG_0, r_var, CG_DIRECT_ALL);
      ```
*TYPICALLY IN NESTED LOOPS:*

Stack is essentially a list that can be declare as
```c  
t_list *execStack = NULL;

/* Push a new ‘exec’ information structure onto the stack. */
      t_exec_operator *blockData = malloc(sizeof(t_exec_operator));
      execStack = addFirst(execStack, blockData);
```
NBNB:
```t_exec_operator``` is a structure that MUST be defined: (into axe_struct.h)

```c
typedef struct t_exec_operator {
   /* Label that points at the end of the block. */
   t_axe_label *l_exit;
   /* Register containing the result of the expression. */
   int r_result;
} t_exec_operator;
```
// the type of each list node is t_exec_operator*
/* Get the context information of the innermost ‘exec’ (it was an operator that allows nesting in an exercise, use it as general name)*/
t_exec_operator *blockData = LDATA(nameList);

/* NB: for int data use LINTDATA, for general type (like this) use LDATA */

The MALLOC has been used to reserve space and blockData refers to that space, it is possible to reserve also registers and labels.
For example:
```c
  blockData -> r_result = gen_load_immediate(program,0); //reserve a register
  blockData -> l_exit = newLabel(program); //reserve a label
```

Now it is time to use and then POP information stored:
```c

      /* Access to reserved label and register */
      assignLabel(program, blockData->l_exit); //access to label
      gen_add_instruction(program, blockData->r_result,...) // save in r_result a datum, it is a reserved register stored in the list.
      gen_bt_instruction(program, blockData->l_exit, 0); //example of using reserved label in the list to jump

      //now POP information
      /* Pop the ‘exec’ information structure from the top of the stack. */
      t_exec_operator *blockData = LDATA(execStack);
      execStack = removeFirst(execStack);
      /* Free the ‘exec’ information structure as we don’t need it anymore */
 free(blockData); // remember to free
```


****

in case of statement the situation changes:

The idea is to use the token as a reference to the struct that will contain information that can be nested :

```c
//this goes in sematic section of acse.y
record %union {
  t_name_statement name_stmp;
}
// and also assign it to the token
%token <name_stmp> nameToken

```
```c
//(This goes into acse_struct.h file)
typedef struct t_exec_operator {
   /* Label that points at the end of the block. */
   t_axe_label *l_exit;
   /* Register containing the result of the expression. */
   int r_result;
} t_exec_operator;

// for example how to assign values:
/* Reserve a register that will buffer the old value of the variable */
  $1.r_oldvalue = getNewRegister(program);
  /* Generate a label that points to the body of the loop */
  $1.l_loop = assignNewLabel(program);
```
****

**CREATING LABEL IN ACSE**

**3 primary functions to do this:**

1. ```newLabel() ```-> creates a label structure without inserting it into the istruction list
code: ```t_axe_label *newLabel(t_program_infos *program);```

  example: ```check = newLabel(program); (check has been defined globally with t_axe_label)```

2. ```assignLabel()``` -> insert the label in the instrcution list;
```void assignLabel(program, t_axe_label *label);```
3. ```assignNewLabel()``` -> creates and insert the label at the same time:
  ```c
  t_axe_label *assignNewLabel(program) {
    t_axe_label label = newLabel(program);
    return assignLabel(program, label);
  }
  ```
**TIPS:**
- forward branch -> new and assign
- backward branch -> assignNewLabel directly at the beginning

**BRANCH CONDITIONS**

When we want to evaluate an expression in order to jump or to do something, it is possible to do that using a direct function.

```gen_andb_instruction(program, val, val, val, CG_DIRECT_ALL);```

This function evaluates different exp:

Valid values for ‘condition’ are:
_LT_
_GT_
_EQ_
_NOTEQ_
_LTEQ_
_GTEQ_

It generates instructions that perform a comparison between two values. It takes in input two expressions and it returns a new expression that represents the result. If the two expressions are both IMMEDIATE, no instructions are generated and IMMEDIATE expression is returned. 0 IF COMPARISON IS FALSE, 1 IF COMPARISON IS TRUE.  

The only way to branch following a condition result is set the system flag before check.
```c
t_axe_expression cmp = handle_binary_comparison(program, iv, $8, _NOTEQ_);
gen_andb_instruction(program, cmp.value, cmp.value, cmp.value, CG_DIRECT_ALL);

//OR

handle_binary_comparison(program,  create_expression(loc, REGISTER), create_expression($7, REGISTER), _LTEQ_);
gen_bne_instruction(program, loop_return, 0);
```

*NOTE ABOUT HOW TO BRANCH*

(NOT SURE IF CORRECT)

When we want to evaluate a condition in order to brach is as well as we want to set a flag and then branch. In order to do so, immediately after the branch condition we have to put the update of the branch condition.

For example

we would like to replicate the for loop from 10 to 0, so, normally it is necessary to branch until index becomes 0.

standard C code:
```c
  for(int i = 10; i > 0; i++){}
   // in ACSE is different (when we need to generate code to do so (like reg see later))

   // register of index i
   int index = getNewRegister(program);

   // first instruction of the loop ( we need to return here )
   t_axe_label *back = assignNewLabel(program); // see kinds of label to understand this

   /* other operations*/

   gen_subi_instruction(program, index, index, 1); // decrease by 1 the index, to but immediately before branch evaluation
   gen_bne_instruction(program, back, 0); // IF INDEX != 0 JUMP TO BACK
   //^^^^^ 0 as third argument doesn't matter, it is always 0
   /* Is the gen_subi_instruction that set the flags == 0 ??? it works but I'm not sure*/


```


*Evaluate conditions for loop like i = i + 1:*

When in a loop we want to increment the variable for example
for(i = 0, i < 10, i = i + 1);
and for some reason we create separate statements in order to create lists of assignements it is useful to load normally the first part as well as the second one. Because if we use for example
```c
int loc = get_symbol_location(program, loc, 0);

if($3.expression_type = IMMEDIATE){
  gen_addi_instruction(program, loc, REG_0, $3.value); ....
}
```

automatically when i = i + 1 is read the loc is incremented by 1 because automatically evaluates the expression i = i + 1;


*****
**ARRAYS**

Load an array element: int val = ```loadArrayElement(program, $4, exp);```

Get infos about the array: ```t_axe_variable *my_arr = getVariable(program, $4);```

Check if an IDENTIFIER is an array:
```c
if(!my_arr->isArray){ <<// array size
            yyerror("This is not an array!");
            YYERROR;
}
```
```
void storeArrayElement(t_program_infos *program, char *ID, t_axe_expression index, t_axe_expression data);
```
NB:
* when you want see the size arr->arraySize store into a register
* when you retreive a value as int val = loadArrayElement... it is not an immediate, it is a REGISTER
****

**Others (expressions)**

Check if a particular statement / exp is empty, for example a optional part in a statement.

firstly declare ```t_axe_expression_opt expr_opt; and %type <expr_opt> exp_opt``` then
```c
exp_opt : exp { $$.expr = $1; $$.empty = 0; }
        | { $$.empty = 1; }
        ;
        ```
Then when we want in the "main statement" see if this is empty or not we can use:
```c
// This allow to check if exp_opt is empty or not and in the case that is full it evaluates direclty the expression/s that it contains.

if (!$6.empty) { //$6 indicates exp_opt
    if ($6.expr.expression_type == IMMEDIATE)
        gen_load_immediate(program, $6.expr.value);
    else
        gen_andb_instruction(program, $6.expr.value, $6.expr.value, $6.expr.value, CG_DIRECT_ALL);

        gen_beq_instruction(program, $1.label_end, 0); // branch if flag is false
}
```

**MANAGE BINARY NUMBERS**

If we have as input a number (exp) and we want to evaluate some operation on his binary representation we can do that using logic operators provided in acse.
At first it is necessary to identify the expression type because the operations that will be performed are a bit different.

(ACSE USES CO2 COMPLEMENT TO RAPRESENT NUMBERS)

```c
  if($3.expression_type == IMMEDIATE)
  /*
  The expression is immediate, we compute the value at compile time.
  Notice that it is not possible to exploit handle_bin_numeric_op
  or handle_binary_operator to handle constant-folding "for free"
  because the implementation requires a loop.
  So it is necessary to create a loop to manage EACH SINGLE BIT the the number represents. (NB: 32 bit max representation in acse, so loop until 32).
  */
  ```c
    int res = 0;
    unsigned int tmp = $3.value; //READ $3.VALUE AS UNSIGNED

    //loop
    for (int i=0; i<32; i++) {
            res += tmp & 1; // it takes least significant bit and do AND with it
            // the idea is to count the numer of ones in this case so an add is performed
            tmp = tmp >> 1; // it takes the tmp and do shift to right so the least significant bit is deleted.
            // thanks to this the cycle it is possible to scan all the bits
         }

  /*NB: if the exp is a register we have to generate code at runtime*/


  ```
^^ BIG DIFFERENT:

 + *expression_type = IMMEDIATE* -> IT IS POSSIBLE TO GENERATE CODE THAT COMPUTES THE VALUE AT *COMPILE TIME* (USE C NORMAL CODE INSIDE)

+ *expression_type = REGISTER* -> MANDATORY TO GENERATE CODE WHICH WILL COMPUTE THE VALUE AT *RUNTIME* -!!!!!> NEED TO GENERATED CODE


** VERY IMPORTANT **

When it is necessary evaluate a branch between expression USE BEFORE CONDITION handle_bin_numeric_op like previous tips; NEVER FORGET THIS
In addition when an expression has to be defined globally use:
``` t_axe_expression nameExpression ```

In addition when it is necessary to create an expression to use in handle bin in order to evalute a condition the expression  must be a REGISTER :
``` create_expression(value, REGISTER) ```

When you want to declare a variable as GLOBAL and the you want to store something into it
BEFORE DO THIS IT IS MANDATORY TO ASSOCIATE A REGISTER TO IT.

```c
// in inst list
int my_value;

//in a exp/statement BEFORE STORE SOMETHING INTO IT
int my_value = getNewRegister(program);
```
^^ This operation is not needed if my_value has been inizialize into variables section using for example a gen_load_immediate.


NB: non so il motivo ma quando valuto se saltare funziona al contrario, se voglio saltare quando index == size, faccio la SUB e poi una BEQ.

(con handle_binary_comparison funziona giusto)!


free when use get_symbol_location or getVariable
