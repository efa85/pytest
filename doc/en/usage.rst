
.. _usage:

Usage and Invocations
==========================================


.. _cmdline:

Calling pytest through ``python -m pytest``
-----------------------------------------------------

.. versionadded:: 2.0

You can invoke testing through the Python interpreter from the command line::

    python -m pytest [...]

This is almost equivalent to invoking the command line script ``pytest [...]``
directly, except that python will also add the current directory to ``sys.path``.

Getting help on version, option names, environment variables
--------------------------------------------------------------

::

    pytest --version   # shows where pytest was imported from
    pytest --fixtures  # show available builtin function arguments
    pytest -h | --help # show help on command line and config file options


Stopping after the first (or N) failures
---------------------------------------------------

To stop the testing process after the first (N) failures::

    pytest -x            # stop after first failure
    pytest --maxfail=2    # stop after two failures

Specifying tests / selecting tests
---------------------------------------------------

Several test run options::

    pytest test_mod.py   # run tests in module
    pytest somepath      # run all tests below somepath
    pytest -k stringexpr # only run tests with names that match the
                          # "string expression", e.g. "MyClass and not method"
                          # will select TestMyClass.test_something
                          # but not TestMyClass.test_method_simple
    pytest test_mod.py::test_func  # only run tests that match the "node ID",
                                    # e.g. "test_mod.py::test_func" will select
                                    # only test_func in test_mod.py
    pytest test_mod.py::TestClass::test_method  # run a single method in
                                                 # a single class

Import 'pkg' and use its filesystem location to find and run tests::

    pytest --pyargs pkg # run all tests found below directory of pkg

Modifying Python traceback printing
----------------------------------------------

Examples for modifying traceback printing::

    pytest --showlocals # show local variables in tracebacks
    pytest -l           # show local variables (shortcut)

    pytest --tb=auto    # (default) 'long' tracebacks for the first and last
                         # entry, but 'short' style for the other entries
    pytest --tb=long    # exhaustive, informative traceback formatting
    pytest --tb=short   # shorter traceback format
    pytest --tb=line    # only one line per failure
    pytest --tb=native  # Python standard library formatting
    pytest --tb=no      # no traceback at all

The ``--full-trace`` causes very long traces to be printed on error (longer
than ``--tb=long``). It also ensures that a stack trace is printed on
**KeyboardInterrupt** (Ctrl+C).
This is very useful if the tests are taking too long and you interrupt them
with Ctrl+C to find out where the tests are *hanging*. By default no output
will be shown (because KeyboardInterrupt is caught by pytest). By using this
option you make sure a trace is shown.

Dropping to PDB_ (Python Debugger) on failures
-----------------------------------------------

.. _PDB: http://docs.python.org/library/pdb.html

Python comes with a builtin Python debugger called PDB_.  ``pytest``
allows one to drop into the PDB_ prompt via a command line option::

    pytest --pdb

This will invoke the Python debugger on every failure.  Often you might
only want to do this for the first failing test to understand a certain
failure situation::

    pytest -x --pdb   # drop to PDB on first failure, then end test session
    pytest --pdb --maxfail=3  # drop to PDB for first three failures

Note that on any failure the exception information is stored on
``sys.last_value``, ``sys.last_type`` and ``sys.last_traceback``. In
interactive use, this allows one to drop into postmortem debugging with
any debug tool. One can also manually access the exception information,
for example::

    >>> import sys
    >>> sys.last_traceback.tb_lineno
    42
    >>> sys.last_value
    AssertionError('assert result == "ok"',)

Setting a breakpoint / aka ``set_trace()``
----------------------------------------------------

If you want to set a breakpoint and enter the ``pdb.set_trace()`` you
can use a helper::

    import pytest
    def test_function():
        ...
        pytest.set_trace()    # invoke PDB debugger and tracing

.. versionadded: 2.0.0

Prior to pytest version 2.0.0 you could only enter PDB_ tracing if you disabled
capturing on the command line via ``pytest -s``. In later versions, pytest
automatically disables its output capture when you enter PDB_ tracing:

* Output capture in other tests is not affected.
* Any prior test output that has already been captured and will be processed as
  such.
* Any later output produced within the same test will not be captured and will
  instead get sent directly to ``sys.stdout``. Note that this holds true even
  for test output occurring after you exit the interactive PDB_ tracing session
  and continue with the regular test run.

.. versionadded: 2.4.0

Since pytest version 2.4.0 you can also use the native Python
``import pdb;pdb.set_trace()`` call to enter PDB_ tracing without having to use
the ``pytest.set_trace()`` wrapper or explicitly disable pytest's output
capturing via ``pytest -s``.

.. _durations:

Profiling test execution duration
-------------------------------------

.. versionadded: 2.2

To get a list of the slowest 10 test durations::

    pytest --durations=10


Creating JUnitXML format files
----------------------------------------------------

To create result files which can be read by Jenkins_ or other Continuous
integration servers, use this invocation::

    pytest --junitxml=path

to create an XML file at ``path``.

record_xml_property
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. versionadded:: 2.8

If you want to log additional information for a test, you can use the
``record_xml_property`` fixture:

.. code-block:: python

    def test_function(record_xml_property):
        record_xml_property("example_key", 1)
        assert 0

This will add an extra property ``example_key="1"`` to the generated
``testcase`` tag:

.. code-block:: xml

    <testcase classname="test_function" file="test_function.py" line="0" name="test_function" time="0.0009">
      <properties>
        <property name="example_key" value="1" />
      </properties>
    </testcase>

.. warning::

    ``record_xml_property`` is an experimental feature, and its interface might be replaced
    by something more powerful and general in future versions. The
    functionality per-se will be kept, however.

    Currently it does not work when used with the ``pytest-xdist`` plugin.

    Also please note that using this feature will break any schema verification.
    This might be a problem when used with some CI servers.

LogXML: add_global_property
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. versionadded:: 3.0

If you want to add a properties node in the testsuite level, which may contains properties that are relevant
to all testcases you can use ``LogXML.add_global_properties``

.. code-block:: python

    import pytest

    @pytest.fixture(scope="session")
    def log_global_env_facts(f):

        if pytest.config.pluginmanager.hasplugin('junitxml'):
            my_junit = getattr(pytest.config, '_xml', None)

        my_junit.add_global_property('ARCH', 'PPC')
        my_junit.add_global_property('STORAGE_TYPE', 'CEPH')

    @pytest.mark.usefixtures(log_global_env_facts)
    def start_and_prepare_env():
        pass

    class TestMe:
        def test_foo(self):
            assert True

This will add a property node below the testsuite node to the generated xml:

.. code-block:: xml

    <testsuite errors="0" failures="0" name="pytest" skips="0" tests="1" time="0.006">
      <properties>
        <property name="ARCH" value="PPC"/>
        <property name="STORAGE_TYPE" value="CEPH"/>
      </properties>
      <testcase classname="test_me.TestMe" file="test_me.py" line="16" name="test_foo" time="0.000243663787842"/>
    </testsuite>

.. warning::

    This is an experimental feature, and its interface might be replaced
    by something more powerful and general in future versions. The
    functionality per-se will be kept.

Creating resultlog format files
----------------------------------------------------

.. deprecated:: 3.0

    This option is rarely used and is scheduled for removal in 4.0.

To create plain-text machine-readable result files you can issue::

    pytest --resultlog=path

and look at the content at the ``path`` location.  Such files are used e.g.
by the `PyPy-test`_ web page to show test results over several revisions.

.. _`PyPy-test`: http://buildbot.pypy.org/summary


Sending test report to online pastebin service
-----------------------------------------------------

**Creating a URL for each test failure**::

    pytest --pastebin=failed

This will submit test run information to a remote Paste service and
provide a URL for each failure.  You may select tests as usual or add
for example ``-x`` if you only want to send one particular failure.

**Creating a URL for a whole test session log**::

    pytest --pastebin=all

Currently only pasting to the http://bpaste.net service is implemented.

Disabling plugins
-----------------

To disable loading specific plugins at invocation time, use the ``-p`` option
together with the prefix ``no:``.

Example: to disable loading the plugin ``doctest``, which is responsible for
executing doctest tests from text files, invoke pytest like this::

    pytest -p no:doctest

.. _`pytest.main-usage`:

Calling pytest from Python code
----------------------------------------------------

.. versionadded:: 2.0

You can invoke ``pytest`` from Python code directly::

    pytest.main()

this acts as if you would call "pytest" from the command line.
It will not raise ``SystemExit`` but return the exitcode instead.
You can pass in options and arguments::

    pytest.main(['-x', 'mytestdir'])

You can specify additional plugins to ``pytest.main``::

    # content of myinvoke.py
    import pytest
    class MyPlugin:
        def pytest_sessionfinish(self):
            print("*** test run reporting finishing")

    pytest.main(["-qq"], plugins=[MyPlugin()])

Running it will show that ``MyPlugin`` was added and its
hook was invoked::

    $ python myinvoke.py
    *** test run reporting finishing
    

.. include:: links.inc
