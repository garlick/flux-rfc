.. github display
   GitHub is NOT the preferred viewer for this file. Please visit
   https://flux-framework.rtfd.io/projects/flux-rfc/en/latest/spec_7.html

7/Flux Coding Style Guide
=========================

This specification presents the recommended standards when contributing code to the Flux code base.

-  Name: github.com/flux-framework/rfc/spec_7.rst

-  Editor: Tom Scogland <scogland1@llnl.gov>

-  Original author: Don Lipari <lipari@llnl.gov>

-  State: draft


Language
--------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to
be interpreted as described in `RFC 2119 <https://tools.ietf.org/html/rfc2119>`__.


Related Standards
-----------------

-  https://www.kernel.org/doc/Documentation/CodingStyle


Goals
-----

-  Encourage a uniform coding style in flux-framework projects

-  Provide a document to reference when providing style feedback in project pull requests


C Coding Style Recommendations
------------------------------

Flux projects written in C SHOULD conform to the C99 version of the language.

In general, Flux follows the "Kernighan & Ritchie coding style" with the following exceptions or examples:

1. Indenting SHALL be with spaces, and not tabs.

2. One level of indentation SHALL be 4 spaces.

3. One space SHALL separate function names and the opening parenthesis (enhances readability).

4. There SHALL be no trailing spaces (or tabs) after the last non-space character in a line.

5. Lines SHOULD be limited to 80 characters.

6. Comments SHOULD be indented to the depth of the code they are describing.

7. The return type SHOULD be on the same line as the function declaration.

8. One space SHOULD separate the star and type in pointer declarations. Example:

::

   int *ptr;


Variable Names
~~~~~~~~~~~~~~

Variable names SHOULD NOT include upper case letters.
For example ``msg_count`` is OK, but ``MsgCount`` or ``MSG_COUNT`` do not conform.

Preprocessor macro names SHOULD NOT include lower case letters.
For example ``FLUX_FOO_MAGIC`` is OK but ``flux_foo_magic`` and ``FluxFooMagic`` do not conform.


Typedefs
~~~~~~~~

C typedef names SHOULD NOT include upper case letters.

C typedef names for functions SHOULD end in ``_f`` and SHOULD be defined as function pointers, for example:

::

   typedef int (*flux_foo_f)(int arg1, int arg2);

C typedef names for non-functions SHOULD end in ``_t``, for example:

::

   typedef int my_type_t;

Abstract types SHOULD be publicly defined as incomplete types, for example:

::

   typedef struct foo foo_t;

Abstract types SHOULD NOT be defined as pointers to incomplete types, as
this obscures the fact that they are pointers when used in declarations.

Typedef names SHOULD be chosen such that they are unique within a framework project.
For example avoid generic types like ``struct a`` or typedefs like ``ctx_t``. A good
practice is to include the name of the relevant source file, module or service in
the typedef (e.g. ``fooservice_ctx_t``).

Typedefs SHOULD NOT be used for fixed length character arrays, as this
obscures the fact that they are merely pointers when used in function
prototypes, and give different ``sizeof`` results depending on context.


Structures
~~~~~~~~~~

Structure tags SHOULD NOT contain the string "struct".

In the same way that we would not write:

.. code:: c

   int count_int;
   float val_float;

we also should not write:

.. code:: c

   struct foo_struct {};


Enums and constants
~~~~~~~~~~~~~~~~~~~

Enumerations SHOULD be used instead of preprocessor macros for integral
constants. Each ``enum`` SHOULD have a name, to facilitate bindings and avoid
casts in stronger typed contexts, for example ``enum flux_<name> {};`` rather than
``enum {};``. If possible, the name should be derived from the function or
structure it is meant to apply to. Further, the name SHOULD have a suffix of
``_b`` if it is a bitmask enum.

Examples:

.. code:: c

   // Good, named, none is not a valid state
   enum flux_msg_type_b {
       FLUX_MSGTYPE_REQUEST    = 0x01,
       FLUX_MSGTYPE_RESPONSE   = 0x02,
       FLUX_MSGTYPE_EVENT      = 0x04,
       FLUX_MSGTYPE_PUP        = 0x08,
       FLUX_MSGTYPE_ANY        = 0x0f,
       FLUX_MSGTYPE_MASK       = 0x0f,
   };

   // Good, named with suffix, bitmap with none valid
   enum flux_flag_b {
       FLUX_O_NONE = 0,
       FLUX_O_TRACE = 1,
       FLUX_O_CLONE = 2,
       FLUX_O_NONBLOCK = 4,
       FLUX_O_MATCHDEBUG = 8,
   };

   // Good, non-bitmap, and none is not a valid state, 0 isn't an option
   enum flux_requeue_mode {
       FLUX_REQUEUE_HEAD = 1,   /* requeue message at head of queue */
       FLUX_REQUEUE_TAIL = 2,   /* requeue message at tail of queue */
       FLUX_REQUEUE_RAND = 3,   /* requeue message at somewhere*/
   };

In order to represent the full range of values, enums that use a zero for none
or similar SHOULD include an item with the value zero to represent that state.


Tools for C formatting
~~~~~~~~~~~~~~~~~~~~~~

The flux-core repository includes a ``.clang-format`` file for use with
clang-format, and SHOULD be used for automated formatting if possible.

Those using vi will automatically follow some of the Flux style based on the presence of the following at the end of each file:

::

   /*
    * vi:tabstop=4 shiftwidth=4 expandtab
    */

In vim, use the following to highlight whitespace errors:

::

   let c_space_errors = 1

In emacs, add this to your custom-set-variables defs to highlight whitespace errors:

::

   '(show-trailing-whitespace t)


Python coding style
-------------------

-  Python code SHALL be formatted with the `Black code style <https://black.readthedocs.io/en/stable/the_black_code_style/index.html>`__.
