It may be helpful to follow the actions and routines that handle a module load.
This will be explained from the outer most level and will work inward.

##Level 0:

To start with a bash user enters a "module load foo".  Let assume that the foo module
contains:

    setenv ("FOO_VERSION","1.0")

and the module command is defined as

    module () { eval $(/path/to/lmod bash "$@"); }

The user command translates to

    eval $( /path/to/lmod bash "load" "foo" )

And the command

    /path/to/lmod/bash "load" "foo"

generates in its simplest form (hiding many details, explained later)

    export FOO_VERSION=1.0;

Which when evaluated sets FOO_VERSION to be "1.0" in the environment.  This is the basic
way that modules have work and has nothing to do with the internals of Lmod.  The main
point here is that the *lmod* command generates text which is a simple shell program which
the module shell function evaluates.  This is how the user's environment has changed
by *loading* the foo module.  

##Level 1:

This level is starts with "/path/to/lmod bash 'load' 'foo'".

The lmod.lua script is the entry point to the Lmod complex.  The main steps are:

    1.  Parse Command line arguments
    2.  Find the action the user requested (e.g. load the foo module).
    3.  Invoke the action.
    4.  Output the changes.

Invoking the action builds two data structures *varTbl* and *MT*.  The *varTbl* is an
array of environment variable that the actions like loading a module will change such
as setting the *FOO_VERSION* variable to "1.0".

The other data structure built is *MT*.  This is the module table.  It is a lua table
and it contains a list of the module loaded, what is the filename for the module and
other information.  This table is the only way Lmod knows the state of the loaded modules
between invocations.

In step 4, Lmod reports the key value pairs in *varTbl* and the current value of *MT*
and generates simple program to be evaluated.  The new key value pairs are written as
bash, csh, python or perl script depending on what the first argument to the lmod
command is.

The module table is a lua table and looks like:

    mt = {
      activeSize=1,
      mT= {
        foo = {
           FN="/path/to/foo/1.0.lua",
           ...
           }
       }
    }

With the mix of quotes and commas it is difficult something like that directly, especially
in C-shell.  So Lmod serializes the table similarily to what is shown above but without
spaces and newlines removed.  Then that text is base64 encoded and stored in variables named
_ModuleTable001_, _ModuleTable002_, etc in blocks of 256 characters.  When Lmod starts, it
grabs those env. vars and puts them together to base64 decodes them to recover the current
state of the modules loaded.  So when Level 0 stated that the export statement was created
by Lmod, it left out that the Module Table was also printed out.

To see the current module table you can do:

   $ module --mt

## Level 2

The command line actions are in src/cmdfuncs.lua.  Similarily, functions specified in module
files are in src/modfuncs.lua.  Explain why they are similar but not the same and maintained
separately.

## Level 3

When a module is loaded, the actions are treated in a positive way.  They generally say

    setenv("ABC","DEF")
    prepend_path("PATH", "/path/to/add")
    
So loading this module would set **ABC** and prepend to the **PATH** variable.  When a module is
unloaded the *setenv* and *prepend\_path* actions are reversed.  So when these modulefiles
are interpreted the action of the functions depends on which mode.  There are several modes
among then are **load**, **unload**, **show** and **help**.

When implementing a module file interpreter, one could have a single *setenv* function which
checked the mode to decide which action to implement (e.g. load, unload, print or no-op).
Instead Lmod uses an object oriented approach by using an factory to build an object which
performs the action in the desired direction.  The program maintains a global variable
*mcp* which first built in lmod.lua:

    mcp = MasterControl.build("load")

This builds an object which places actions in the positive.  When a user requests an unload,
then the program builds:

    mcp = MasterControl.build("unload")

This places the action in the negative direction.  An *setenv* or *prepend\_path* will unset
a global variable or remove an entry from a path. 

Another way that module files are interpreted is for show.  A *mcp* is built by:

    mcp = MasterControl.build("show")

This changes the modulefile actions to converted into print statements.  There are several
other modes that treat the actions of modulefiles differently.  The way that this is
implemented is that there is base class MasterControl (in src/MasterControl.lua) which
implements the functions a single positive and another in negative directions.  There is
a member load function and another member unload function.  When building the **load** version
of *mcp* the member setenv function points to the MasterControl.setenv function.  This can
be seen in src/MC_Load.lua

When building the **unload** version the setenv member function points to
MasterControl.unsetenv function as seen in src/MC_Unload.lua.  It is the construction
of the **load**, **unload**, **show**, etc versions of *mcp*  that changes the
interpretation of the modulefile function.

## Level 4

Explain MName and the load and prereq modifiers work MN_*.

## Level 5

Explain how the spider cache works.

## Level 6

Explain how the sandbox works and why it is important.

## Other possible topics

*Master.lua
*How Lmod supports both sending messages stderr but will support sending module list, avail,
  etc to stdout.
*hooks?
* 





