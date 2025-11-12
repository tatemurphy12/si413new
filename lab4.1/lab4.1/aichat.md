
Given the token specifications and language parse rules, how could I allow for functions to be treated as first class citizens? ex: function defs within function defs and functions returning other functions

This is a great question, as it gets to the heart of creating a more powerful and flexible language.

To treat functions as first-class citizens (passing them as arguments, returning them, and defining them within other functions), you need to make several key changes to your grammar.

First, the good news: Your current grammar already supports function definitions within function definitions (nested functions).

This is because your DEF rules [Source 6] contain a stmtList: DEF ... FUNC ... OPENEX stmtList ... CLOSEX

And stmtList can contain any stmt [Source 5], which in turn includes the DEF rules [Source 6]. So, you can already nest a define inside another define.

The real challenge is passing and returning functions. Your current grammar can't do this because of a major blocker in your token specification.

￼ The Core Problem: Your FUNC Token
Your FUNC token is defined as ([A-Za-z_][A-Za-z0-9-]+\() [Source 3].

This token is "greedy." It matches both the function's name and its opening parenthesis (. This makes it impossible to refer to a function by its name as a value (e.g., send myFunc). The lexer will only recognize myFunc( as a FUNC token.

To fix this, you must separate the name from the (. The standard way to do this is to have a single ID (identifier) token for all names (variables and functions) and let the parser figure out the context.

How to Fix Your Grammar (Step-by-Step)
Here is the strategy to implement first-class functions.

1. Fix Your tokenSpec.txt
You need to stop differentiating between VAR and FUNC in the lexer. Let them all be one "identifier" token. You also need a new token for the "function" type.

Recommended tokenSpec.txt changes:

// ... (keep ASSIGN, PRINT, etc.)
TYPESTR:       string
TYPEBOOL:      bool
TYPEVOID:   void
TYPEFUNC:      function  // <-- NEW: Add a type for functions

DEF:   define
RETURN:     send
// ... (keep WHILE, IF, etc.)

// --- IMPORTANT CHANGES ---
// REMOVE these:
// VAR:           [a-zA-Z0-9]*$       (This regex is also broken)
// FUNC:     ([A-Za-z_][A-Za-z0-9-]+\()

// ADD this:
ID:            [a-zA-Z_][a-zA-Z0-9_]* // <-- NEW: A single token for all identifiers

// ... (keep rest)
2. Update ParseRules.g4 with New Tokens
First, update the tokens block to use ID and TYPEFUNC, and remove VAR and FUNC.

Code snippet
// Add TYPEFUNC, replace VAR and FUNC with ID
tokens { ASSIGN, PRINT, TYPESTR, TYPEBOOL, TYPEVOID, TYPEFUNC, DEF, RETURN,
         WHILE, IF, CONCAT, REV, IN, CON, LT, AND, OR, NOT, OPENEX,
         CLOSEX, BOOL, STR, ID }
3. Update All Rules to Use ID
Now, you must change every rule that used VAR or FUNC to use ID instead. Crucially, you must also add the OPENEX and CLOSEX tokens that FUNC used to "hide."

Example: stmt rules [Source 6]

Code snippet
stmt
  : ASSIGN OPENEX TYPESTR ID strEx CLOSEX #AssignStr  // VAR -> ID
  : ASSIGN OPENEX TYPEBOOL ID boolEx CLOSEX #AssignBool // VAR -> ID
  : PRINT OPENEX TYPESTR strEx CLOSEX #PrintStr
  : PRINT OPENEX TYPEBOOL boolEx CLOSEX #PrintBool
  : IF OPENEX boolEx stmtList stmtList CLOSEX #IfElse
  : WHILE OPENEX boolEx stmtList CLOSEX #WhileLoop
 
  // Rules for function definitions (FUNC -> ID, added OPENEX)
  : DEF TYPESTR ID OPENEX paramList CLOSEX OPENEX stmtList RETURN strEx CLOSEX #DefStrFun
  : DEF TYPEBOOL ID OPENEX paramList CLOSEX OPENEX stmtList RETURN boolEx CLOSEX #DefBoolFun
  : DEF TYPEVOID ID OPENEX paramList CLOSEX OPENEX stmtList CLOSEX #DefVoidFun
 
  // Rule for void function call (FUNC -> ID, added OPENEX)
  : ID OPENEX argList CLOSEX #VoidFunCall
  // Note: I removed TYPEVOID, as the call itself is void,
  // you don't need to specify its type when calling.
;
Example: param and arg rules [Source 8, 10]

Code snippet
paramList
  : param paramList #RegParam
  :                 #EmptyParam
  ;
param
  : TYPEBOOL ID     // VAR -> ID
  : TYPESTR ID      // VAR -> ID, fixed TYPESTRING typo
  | TYPEFUNC ID     // <-- NEW: Allow function-type parameters
  ;

argList
  : arg argList #RegArg
  : #EmptyArg
  ;
arg
  : STR #StrLitArg
  : BOOL #BoolLitArg
  : ID #VarArg      // <-- SIMPLIFIED: Just pass an identifier
  ;
  // Note: The old rules (e.g., TYPEBOOL VAR) were redundant.
  // The type of the variable is known by the compiler, not needed at the call site.
Example: strEx and boolEx calls [Source 11, 12]

Code snippet
strEx
  : // ... (other rules)
  : ID OPENEX argList CLOSEX #StrFunCall // <-- Use ID, add OPENEX
  | ID #StrVar                          // VAR -> ID
  ;
 
boolEx
  : // ... (other rules)
  : ID OPENEX argList CLOSEX #BoolFunCall // <-- Use ID, add OPENEX
  | ID #BoolVar                          // VAR -> ID
  ;
4. Add Rules for First-Class Functions (The Goal)
Now that the grammar is cleaned up, you can add the rules to make functions first-class.

Code snippet
// Add to your parser grammar (e.g., after boolEx)

// A "function expression" is a value that IS a function
funcEx
  : ID #FuncVar     // A reference to a function (e.g., 'send myFunc')
  | ID OPENEX argList CLOSEX #FuncFunCall // A call that RETURNS a function
  ;

// --- Now, update the rest of the grammar to use TYPEFUNC and funcEx ---

// Add to 'stmt' [Source 6]
stmt
  : // ... (all other stmt rules)
  | ASSIGN OPENEX TYPEFUNC ID funcEx CLOSEX #AssignFunc // Assign a function to a var
  | DEF TYPEFUNC ID OPENEX paramList CLOSEX OPENEX stmtList RETURN funcEx CLOSEX #DefFuncFun // Define a function that returns a function
  ;

// Add to 'arg' [Source 10]
arg
  : STR #StrLitArg
  : BOOL #BoolLitArg
  : ID #VarArg        // This ID could now be a var holding a string, bool, OR function
  ;

// Add to 'strEx', 'boolEx' [Source 11, 12]
// This allows you to call a function stored in a variable
strEx
  : // ...
  | ID OPENEX argList CLOSEX #StrFunCall  // Call by name OR call by variable
  ;

boolEx
  : // ...
  : ID OPENEX argList CLOSEX #BoolFunCall // Call by name OR call by variable
  ;
Summary of Changes
Fixed Tokens: Replaced VAR and FUNC with a single ID token. This is the most critical change.

Added TYPEFUNC: Created a new type keyword, function.

Updated Grammar: Changed all VAR/FUNC rules to use ID and added the necessary OPENEX/CLOSEX tokens.

Generalized Rules:

param can now accept TYPEFUNC ID.

DEF can now define a TYPEFUNC function that RETURNs a funcEx.

ASSIGN can now assign a funcEx to a TYPEFUNC variable.

Simplified arg: An argument is now just an ID (or a literal). Your interpreter/compiler will be responsible for knowing what that ID refers to (a bool, a string, or a function).

Unified Function Calls: The rule ID OPENEX argList CLOSEX now handles both static calls (myFunc(...)) and first-class calls (varHoldingFunc(...)).

Where do I need to specify the type for function calls and IDs?

That's an excellent question, as it gets to the core of why the grammar changes I suggested are so important.

The short answer is: You should only specify a type when you declare (i.e., create or define) a variable or function. You do not need to specify the type when you use (i.e., reference or call) it.

Your old grammar  forced you to specify the type on usage, which is redundant. The compiler or interpreter should be smart enough to remember the type after it's been declared.

Here’s a breakdown of where types belong in the new, cleaner grammar:

For IDs (Variables)
￼ WHERE you DO specify the type:

When you assign it for the first time: Your AssignStr and AssignBool rules  are the perfect example of a declaration. This is where the type is specified.

ASSIGN OPENEX TYPESTR myVar ... CLOSEX

ASSIGN OPENEX TYPEBOOL myVar ... CLOSEX


When you list it as a function parameter: Your param rules  also declare variables, so they must have types.

param: TYPEBOOL myVar

param: TYPESTRING myVar

￼ WHERE you DO NOT specify the type:

When using it in an expression: In your old strEx and boolEx, you had TYPESTR VAR and TYPEBOOL VAR. In the new grammar, this is simplified to just ID.


Old: TYPESTR VAR

New: ID (e.g., #StrVar)


When passing it as an argument: In your old argList, you had rules like TYPEBOOL VAR and TYPESTR VAR. In the new grammar, an arg is simplified to just ID.


Old: arg: TYPEBOOL VAR

New: arg: ID

Why? Once you declare ASSIGN OPENEX TYPESTR myVar ... CLOSEX, the compiler knows myVar is a string. You should never have to specify TYPESTR for myVar again.

For Function Calls
￼ WHERE you DO specify the type:
When you DEFINE the function: This is the function's declaration. The type you specify here is its return type.


DEF TYPESTR myFunc ...  (This function returns a string)


DEF TYPEBOOL myFunc ...  (This function returns a bool)


DEF TYPEVOID myFunc ...  (This function returns void)

￼ WHERE you DO NOT specify the type:

When you CALL the function: In your old grammar, you had rules like TYPESTR FUNC OPENEX argList CLOSEX. This is redundant. The compiler already knows myFunc returns a string because of its DEF rule.


Old: TYPESTR FUNC OPENEX argList CLOSEX

New: ID OPENEX argList CLOSEX (e.g., #StrFunCall)

Why? The parser doesn't need to know what type a function returns; it only needs to know the syntax of a function call. In the new grammar, myStringFunc(arg1) and myBoolFunc(arg2) are both parsed using the exact same rule: ID OPENEX argList CLOSEX. It is the interpreter's (or type-checker's) job to handle the "string" or "bool" that comes back.

A Special Case: Your PRINT Rule
I noticed your PRINT rules are an exception:

PRINT OPENEXTYPESTR strEx CLOSEX

PRINT OPENEX TYPEBOOL boolEx CLOSEX

Here, you are specifying the type at the "call site." This is a valid design choice! It means print isn't a single generic function but two distinct, built-in statements: "print-string" and "print-bool."

If you wanted to, you could simplify this as well: PRINT OPENEX strEx CLOSEX PRINT OPENEX boolEx CLOSEX

This would make print behave more like a standard function call, where the type of its argument (strEx or boolEx) is inferred rather than explicitly stated. This is entirely up to how you want your language to feel.

Did I implement the changes correctly?

Yes, you have correctly implemented the core logic for first-class functions. Your grammar now supports defining, assigning, and returning functions.

You've successfully:

Replaced VAR and FUNC with a single ID token.

Added the TYPEFUNC token and used it in ASSIGN , DEF , and param  rules.

Created a funcEx (function expression) rule  to handle function values.

Updated strEx and boolEx  to use ID for both variables and function calls.

This is a huge step forward. However, there are two critical errors you need to fix, one major inconsistency, and a couple of minor typos.

￼ Critical Errors (Must Fix)
TYPESTRING Typo in param In your param rules, you wrote TYPESTRING ID. Your token is defined as TYPESTR. This will cause your parser to fail.

Fix: Change TYPESTRING to TYPESTR.

Code snippet
param
  : TYPEBOOL ID #BoolParam
  : TYPESTR ID #StrParam  // <-- Fixed
  : TYPEFUNC ID #FuncParam
  ;
Incorrect ID Regex Your ID token is ^[A-Za-z0-9_]+$. The ^ (start of line) and $ (end of line) anchors are a major problem. This regex will only match if the entire line contains just the ID.

Example: assign ( string myVar ... ) will fail because myVar is not at the start of the line.

Fix: Remove the anchors. A standard regex for an ID (that must start with a letter or underscore) is: [a-zA-Z_][a-zA-Z0-9_]*

// In tokenSpec.txt
ID:            [a-zA-Z_][a-zA-Z0-9_]* // <-- Correct regex
￼ Major Inconsistency (Breaks First-Class Calls)
Your VoidFunCall rule  breaks the "first-class" logic you just added.


The Problem: Your rule is TYPEVOID ID OPENEX argList CLOSEX. This only allows you to call a void function by its name (its ID).

Why it's a Problem: What if you assign a void function to a variable?

// This is now legal:
assign ( function myFuncVar myVoidFunction )

// You have no way to call myFuncVar!
// myFuncVar(arg) is not TYPEVOID ID (...)

The Fix: You must make your void function call rule look just like your StrFunCall and BoolFunCall. It should be a statement that is just an ID followed by (...).


Recommended Change to stmt:

Code snippet
stmt
  : ASSIGN OPENEX TYPESTR ID strEx CLOSEX #AssignStr
  : ASSIGN OPENEX TYPEBOOL ID boolEx CLOSEX #AssignBool
  : ASSIGN OPENEX TYPEFUNC ID funcEx CLOSEX #AssignFun
  : PRINT OPENEXTYPESTR strEx CLOSEX #PrintStr
  : PRINT OPENEX TYPEBOOL boolEx CLOSEX #PrintBool
  : IF OPENEX boolEx stmtList stmtList CLOSEX #IfElse
  : WHILE OPENEX boolEx stmtList CLOSEX #WhileLoop
  : DEF TYPESTR ID OPENEX paramList CLOSEX OPENEX stmtList RETURN strEx CLOSEX #DefStrFun
  : DEF TYPEBOOL ID OPENEX paramList CLOSEX OPENEX stmtList RETURN boolEx CLOSEX #DefBoolFun
  : DEF TYPEVOID ID OPENEX paramList CLOSEX OPENEX stmtList CLOSEX #DefVoidFun
  : DEF TYPEFUNC ID OPENEX paramList CLOSEX OPENEX stmtList RETURN funcEx CLOSEX #DefFunFun

  // --- FIXED RULE ---
  : ID OPENEX argList CLOSEX #FunCallStmt  // <-- This REPLACES 'TYPEVOID ID ...'
  ;
This single rule #FunCallStmt now correctly handles calls to void functions (myVoidFunc()) and calls to void functions stored in variables (myFuncVar()), making them truly first-class.

Minor Typos
In tokenSpec.txt: You have BOOl  (with a lowercase 'o'). It should be BOOL.

In ParseRules.g4: Your tokens list has ID twice. This is harmless but unnecessary.

How do I handle variable scope in programs where there is a function def within a function def

This is one of the most important (and fun) parts of building a language. The concept you are describing is called Lexical Scoping (or Static Scoping).

The rule is simple: a function's scope is "enclosed" by the scope of the function that defines it. When looking for a variable, the interpreter follows a "scope chain":

Look in the current, local scope.

If not found, look in the immediate outer (enclosing) scope.

Keep walking up the chain of parent scopes.

If it's not found all the way up to the global scope, the variable is undefined.

￼ How to Implement Lexical Scoping
You cannot use a single, global symbol table. You need to create a chain of environments (or "scopes"). The most common way to do this is to create an Environment object.

Each Environment object must contain two things:

A HashMap<String, Value> for its own local variables.

A pointer to its parent (enclosing) Environment.

Here is the step-by-step logic.

1. The Environment Class
You'll need a class (in Java, Python, C++, etc.) that looks something like this:

class Environment {
    // 1. Pointer to the parent scope
    Environment parent;
   
    // 2. This scope's local variables
    Map<String, Value> localVariables = new HashMap<>();

    // Constructor for global scope
    Environment() {
        this.parent = null;
    }

    // Constructor for a new, nested scope
    Environment(Environment parent) {
        this.parent = parent;
    }

    // Define a NEW variable IN THIS SCOPE (for ASSIGN)
    public void define(String name, Value value) {
        localVariables.put(name, value);
    }

    // Find a variable by walking UP the chain
    public Value get(String name) {
        // 1. Check local scope first
        if (localVariables.containsKey(name)) {
            return localVariables.get(name);
        }
       
        // 2. If not local, check parent
        if (parent != null) {
            return parent.get(name); // Recursive call
        }
       
        // 3. Reached global scope and not found
        throw new Error("Variable '" + name + "' not found.");
    }
}
2. The "Closure": Handling First-Class Functions
This is the critical part. You asked about first-class functions. When you define an inner function, it needs to "remember" the scope it was created in.

When your interpreter executes a DEF statement, it should not just save the code. It must create a Closure object.

A Closure object contains:

The function's code (e.g., the paramList and stmtList from your parse tree).

A pointer to the Environment in which it was defined.

class Closure {
    ParseRules.StmtListContext functionBody;
    ParseRules.ParamListContext parameters;
    Environment capturedEnvironment; // The "birth" environment

    Closure(ParseRules.StmtListContext body, ParseRules.ParamListContext params, Environment env) {
        this.functionBody = body;
        this.parameters = params;
        this.capturedEnvironment = env;
    }
}
3. Putting It All Together: The Walkthrough
Let's trace how your interpreter (e.g., your visitor) would handle this.

Start: Your interpreter has one Environment object: globalEnv. You set currentEnv = globalEnv.

DEF TYPESTR myOuterFunc(...)
You are in globalEnv.

You create a Closure for myOuterFunc.

This Closure contains:

Code: The stmtList for myOuterFunc.

Environment: globalEnv.

You store this Closure in globalEnv with the name "myOuterFunc".

globalEnv.define("myOuterFunc", new Closure(...));

Entering a Function Call: myOuterFunc()
Create a New Scope:

Get the myOuterFunc Closure.

Create a newEnv for the function call: Environment callEnv = new Environment(myOuterFunc.closure.capturedEnvironment);

In this case, callEnv.parent is globalEnv.

Set currentEnv:

currentEnv = callEnv;

Process Parameters: Add the arguments to callEnv.

Execute Body: Visit the stmtList of myOuterFunc within this callEnv.

Nested Definition: DEF TYPESTR myInnerFunc(...)
...While executing myOuterFunc, you hit the DEF for myInnerFunc...

You are now inside callEnv (the scope for myOuterFunc).

You create a Closure for myInnerFunc.

This Closure contains:

Code: The stmtList for myInnerFunc.

Environment: currentEnv (which is callEnv).

You store this new Closure in the current scope:

currentEnv.define("myInnerFunc", new Closure(...));

Nested Function Call: myInnerFunc()
Create a New Scope:

Get the myInnerFunc Closure.

Create a newInnerEnv: Environment newInnerEnv = new Environment(myInnerFunc.closure.capturedEnvironment);

newInnerEnv.parent is now callEnv (the scope of myOuterFunc).

Set currentEnv:

currentEnv = newInnerEnv;

Execute Body: Visit the stmtList of myInnerFunc.

Variable Lookup: (e.g., x) from inside myInnerFunc
...Your code inside myInnerFunc references x, which was a parameter to myOuterFunc...

The interpreter calls currentEnv.get("x") (where currentEnv is newInnerEnv).

newInnerEnv.get("x"):

Looks in newInnerEnv.localVariables. Not found.

Calls parent.get("x"). The parent is callEnv.

callEnv.get("x"):

Looks in callEnv.localVariables (where myOuterFunc's parameters are). Found!

It returns the value of x.

Exiting a Function Call
When your interpreter finishes a stmtList, it must restore the previous environment.

currentEnv = currentEnv.parent;

This simple line automatically "discards" the function's local scope and moves you back up the chain.
