.. _advanced topics:

Advanced Topics
===============

Obfuscating Python Scripts In Different Modes
---------------------------------------------

.. _obfuscating code mode:

Obfuscating Code Mode
~~~~~~~~~~~~~~~~~~~~~
In a python module file, generally there are many functions, each
function has its code object.

* obf_code == 0

The code object of each function will keep it as it is.

* obf_code == 1

In this case, the code object of each function will be obfuscated in
different ways depending on wrap mode.

.. _wrap mode:

Wrap Mode
~~~~~~~~~
* wrap_mode == 0

When wrap mode is off, the code object of each function will be
obfuscated as this form::

    0   JUMP_ABSOLUTE            n = 3 + len(bytecode)

    3    ...
         ... Here it's obfuscated bytecode of original function
         ...

    n   LOAD_GLOBAL              ? (__armor__)
    n+3 CALL_FUNCTION            0
    n+6 POP_TOP
    n+7 JUMP_ABSOLUTE            0

When this code object is called first time

1. First op is JUMP_ABSOLUTE, it will jump to offset n

2. At offset n, the instruction is to call PyCFunction
   `__armor__`. This function will restore those obfuscated bytecode
   between offset 3 and n, and move the original bytecode at offset 0

3. After function call, the last instruction is to jump to
   offset 0. The really bytecode now is executed.

After the first call, this function is same as the original one.

* wrap_mode == 1

When wrap mode is on, the code object of each function will be wrapped
with `try...finally` block::

    LOAD_GLOBALS    N (__armor_enter__)     N = length of co_consts
    CALL_FUNCTION   0
    POP_TOP
    SETUP_FINALLY   X (jump to wrap footer) X = size of original byte code

    Here it's obfuscated bytecode of original function

    LOAD_GLOBALS    N + 1 (__armor_exit__)
    CALL_FUNCTION   0
    POP_TOP
    END_FINALLY

When this code object is called each time

1. `__armor_enter__` will restore the obfuscated bytecode

2. Execute the real function code

3. In the final block, `__armor_exit__` will obfuscate bytecode again.

.. _obfuscating module mode:

Obfuscating module Mode
~~~~~~~~~~~~~~~~~~~~~~~
* obf_mod == 1

The final obfuscated scripts would like this::

    __pyarmor__(__name__, __file__, b'\x02\x0a...', 1)

The third parameter is serialized code object of the Python
script. It's generated by this way::

    PyObject *co = Py_CompileString( source, filename, Py_file_input );
    obfuscate_each_function_in_module( co, obf_mode );
    char *original_code = marshal.dumps( co );
    char *obfuscated_code = obfuscate_algorithm( original_code  );
    sprintf( buffer, "__pyarmor__(__name__, __file__, b'%s', 1)", obfuscated_code );

* obf_mod == 0

In this mode, keep the serialized module as it is::

    sprintf( buffer, "__pyarmor__(__name__, __file__, b'%s', 0)", original_code );

And the final obfuscated scripts would be::

    __pyarmor__(__name__, __file__, b'\x02\x0a...', 0)

Refer to :ref:`Obfuscating Scripts With Different Modes`

.. _restrict mode:

Restrict Mode
-------------

From PyArmor 5.2, Restrict Mode is default setting. In restrict mode,
obfuscated scripts must be one of the following formats::

    __pyarmor__(__name__, __file__, b'...')

    Or

    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    __pyarmor__(__name__, __file__, b'...')

    Or

    from pytransform import pyarmor_runtime
    pyarmor_runtime('...')
    __pyarmor__(__name__, __file__, b'...')

And obfuscated script must be imported from obfuscated script. No any
other statement can be inserted into obfuscated scripts.

For examples, it works::

    $ cat a.py
    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    __pyarmor__(__name__, __file__, b'...')

    $ python a.py

It doesn't work, because there is an extra code "print"::

    $ cat b.py
    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    __pyarmor__(__name__, __file__, b'...')
    print(__name__)

    $ python b.py

It works, import obfuscated script "c.py" from obfuscated script
"d.py"::

    $ cat d.py
    import c
    c.hello()

    # Then obfuscate d.py
    $ cat d.py
    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    __pyarmor__(__name__, __file__, b'...')


    $ python d.py

It doesn't work, because obfuscated script "c.py" can NOT be imported
from no obfuscated scripts in restrict mode::

    $ cat c.py
    __pyarmor__(__name__, __file__, b'...')

    $ cat main.py
    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    import c

    $ python main.py

So restrict mode can avoid obfuscated scripts observed from no
obfuscated code.

Sometimes restrict mode is not suitable, for example, a package used
by other scripts. Other clear scripts can not import obfuscated
package in restrict mode. So it need to disable restrict mode::

    pyarmor obfuscate --restrict=0 foo.py

Besides, if the scripts is obfuscated without restrict mode, you
should disable restrict mode either when generating new licenses for
it::

    pyarmor licenses --restrict=0 --expired 2019-01-01 mycode

.. customizing protection code:

.. include:: _common_definitions.txt
