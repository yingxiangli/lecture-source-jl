.. _testing:

.. include:: /_static/includes/header.raw

***************************************************
Packages, Testing, and Continuous Integration
***************************************************

.. contents:: :depth: 2

.. epigraph::

    A complex system that works is invariably found to have evolved from a simple system that worked. The inverse proposition also appears to be true: A complex system designed from scratch never works and cannot be made to work. You have to start over, beginning with a working simple system -- `Gall's Law <https://en.wikipedia.org/wiki/John_Gall_(author)#Gall%27s_law>`_


Co-authored with Arnav Sood

This lecture discusses structuring a project as a Julia module, and testing it with tools from GitHub

Benefits include

* Specifying dependencies (and their versions) so that your project works across Julia setups and over time

* Being able to load your project's functions from outside without copy/pasting

* Writing tests that run locally, *and automatically on the GitHub server*

* Having GitHub test your project across operating systems, Julia versions, etc.

.. _project_setup:

Project Setup
=======================

Account Setup
--------------------

Travis CI
^^^^^^^^^^^^^

As we'll see later, Travis is a service that automatically tests your project on the GitHub server

First, we need to make sure that your GitHub account is set up with Travis CI and Codecov

As a reminder, make sure you signed up for the GitHub `Student Developer Pack <https://education.github.com/pack/>`_ or `Academic Plan <https://help.github.com/articles/applying-for-an-academic-research-discount/>`_ if eligible

Navigate to the `travis-ci.com website <https://travis-ci.com/>`_ and click "sign up with GitHub" -- supply your credentials

If you get stuck, see the `Travis tutorial <https://docs.travis-ci.com/user/tutorial/>`_

Codecov
^^^^^^^^^

Codecov is a service that tells you how comprehensive your tests are (i.e., how much of your code is actually tested)

To sign up, visit the `Codecov website <http://codecov.io/>`_, and click "sign up"

.. figure:: /_static/figures/codecov-1.png
    :width: 100%

Next, click "add a repository" and *enable private scope* (this allows Codecov to service your private projects)

The result should be

.. figure:: /_static/figures/codecov-2.png
    :width: 100%

This is all we need for now

Julia Setup
--------------------

.. literalinclude:: /_static/includes/deps_generic.jl
     :class: hide-output

**Note:** Before these steps, make sure that you've either completed the :doc:`version control <../more_julia/version_control>` lecture or run

.. code-block:: julia
    :class: no-execute

    git config --global user.name "Your Name"

**Note:** Throughout this lecture, important points and sequential workflow steps are listed as bullets

To set up a project on Julia:

* Load the `PkgTemplates <https://github.com/invenia/PkgTemplates.jl/>`_ package

.. code-block:: julia
    :class: no-execute

    using PkgTemplates

* Create a *template* for your project

This specifies metadata like the license we'll be using (MIT by default), the location (``~/.julia/dev`` by default), etc.

.. code-block:: julia
    :class: no-execute

    ourTemplate = Template(;user="quanteconuser", plugins = [TravisCI(), Codecov()])

**Note**: Make sure you replace the ``quanteconuser`` with your GitHub ID

* Create a specific project based off this template

.. code-block:: julia
    :class: no-execute

    generate("ExamplePackage.jl", ourTemplate)

If we navigate to the package directory, we should see something like the following

.. figure:: /_static/figures/testing-dir.png
    :width: 100%

As a reminder, the location of your ``.julia`` folder can be found by running ``DEPOT_PATH[1]`` in a REPL 

**Note:** On Mac, this may be hidden; you can either start a terminal, ``cd ~`` and then ``cd .julia``, or make `hidden files visible <https://ianlunn.co.uk/articles/quickly-showhide-hidden-files-mac-os-x-mavericks/>`_ in the Finder

Adding a Project to Git
------------------------

The next step is to add this project to Git version control

* Open the repository screen in your account as discussed previously

We'll want the following settings

.. figure:: /_static/figures/testing-git1.png
    :width: 100%

In particular

* The repo you create should have the same name as the project we added

* We should leave the boxes unchecked for the ``README.md``, ``LICENSE``, and ``.gitignore``, since these are handled by ``PkgTemplates``

Then,

* Drag and drop your folder from your ``~/.julia/dev`` directory to GitHub Desktop

* Click the "publish branch" button to upload your files to GitHub

If you navigate to your git repo (ours is `here <https://github.com/quanteconuser/ExamplePackage.jl/>`__), you should see something like

.. figure:: /_static/figures/testing-git2.png
    :width: 100%

**Note:** Be sure that you don't separately clone the repo you just added to another location (i.e., to your desktop)

A key note is that you have some set of files on your local machine (here in ``~/.julia/dev/ExamplePackage.jl``) and git is plugged into those files

For convenience, you might want to create a shortcut to that location somewhere accessible

Adding a Project to the Julia Package Manager
-----------------------------------------------

We also want Julia's package manager to be aware of the project

* Open a REPL in the newly created project directory, either by noting the path printed above, or by running the following in a REPL

.. code-block:: julia
    :class: no-execute

    cd(joinpath(DEPOT_PATH[1], "dev", "ExamplePackage"))

Note the lack of ``.jl``!

* Run the following

.. code-block:: julia
    :class: no-execute

    ] activate

to get into the main Julia environment (more on environments in the second half of this lecture)

* And run

.. code-block:: julia
    :class: no-execute

    ] dev .

to add the package

You can see the change reflected in our default package list by running

.. code-block:: julia
    :class: no-execute

    ] st

For more on the package mode, see the :doc:`tools and editors <../more_julia/tools_editors>` lecture

Using the Package Manager
--------------------------------

Now, from any Julia terminal in the future, we can run

.. code-block:: none

    using ExamplePackage

To use its exported functions

We can also get the path to this by running

.. code-block:: none

    using ExamplePackage
    pathof(ExamplePackage) # returns path to src/ExamplePackage.jl

Project Structure
==========================

Let's unpack the structure of the generated project

* The first directory, ``.git``, holds the version control information

* The ``src`` directory contains the project's source code -- it should contain only one file (``ExamplePackage.jl``), which reads

  .. code-block:: none

      module ExamplePackage

      greet() = print("Hello World!")

      end # module

* Likewise, the ``test`` directory should have only one file (``runtests.jl``), which reads

  .. code-block:: none

      using ExamplePackage
      using Test

      @testset "ExamplePackage.jl" begin
          # Write your own tests here.
      end

In particular, the workflow is to export objects we want to test (``using ExamplePackage``), and test them using Julia's ``Test`` module

The other important text files for now are

* ``Project.toml`` and ``Manifest.toml``, which contain dependency information

In particular, the ``Project.toml`` contains a list of dependencies, and the ``Manifest.toml`` specifies their exact versions and sub-dependencies

* The ``.gitignore`` file (which may display as an untitled file), which contains files and paths for ``git`` to ignore

Project Workflow
=========================

Dependency Management
----------------------------

Environments
^^^^^^^^^^^^^^^^

As :ref:`before <jl_packages>`, the `.toml` files define an *environment* for our project, or a set of files which represent the dependency information

The actual files are written in the `TOML language <https://github.com/toml-lang/toml>`_, which is a lightweight format to specify configuration options

This information is the name of every package we depend on, along with the exact versions of those packages

This information (in practice, the result of package operations we execute) will
be reflected in our ``ExamplePackage.jl`` directory's TOML, once that environment is activated (selected)

This allows us to share the project with others, who can exactly reproduce the state used to build and test it

See the `Pkg3 docs <https://docs.julialang.org/en/v1/stdlib/Pkg/>`_ for more information

Pkg Operations
^^^^^^^^^^^^^^^^^^^

For now, let's just try adding a dependency

* Activate the package environment (to be run from the base, ``v1.1`` environment)

.. code-block:: julia   
    :class: no-execute

    ] activate ExamplePackage

This tells Julia to write the results of package operations to ``ExampleProject``'s TOML,
  and use the versions of packages specified there

Note that the base environment isn't special, except that it's what's loaded by a freshly-started REPL or Jupyter notebook

* Add a package

.. code-block:: julia
    :class: no-execute

    ] add Expectations

We can track changes in the TOML, as before

Here's the ``Manifest.toml``

.. figure:: /_static/figures/testing-atom-manifest.png
    :width: 100%

We can also run other operations, like ``] up``, ``] precompile``, etc.

Package operations are listed in detail in the :doc:`tools and editors <../more_julia/tools_editors>` lecture

Recall that, to quit the active environment and return to the base ``(v1.1)``, simply run 

.. code-block:: julia
    :class: no-execute

    ] activate


Writing Code
-----------------

The basic idea is to work in ``tests/runtests.jl``, while reproducible functions should go in the ``src/ExamplePackage.jl``

For example, let's say we add ``Distributions.jl``

.. code-block:: julia
    :class: no-execute

    ] activate ExamplePackage

.. code-block:: julia
    :class: no-execute

    ] add Distributions

and edit the source (paste this into the file itself ) to read as follows

.. code-block:: julia
    :class: no-execute

    module ExamplePackage

    greet() = print("Hello World!")

    using Expectations, Distributions

    function foo(μ = 1., σ = 2.)
        d = Normal(μ, σ)
        E = expectation(d)
        return E(x -> sin(x))
    end

    export foo

    end # module

Let's try calling this

.. code-block:: julia
    :class: no-execute

    ] activate

.. code-block:: julia
    :class: no-execute

    using ExamplePackage
    ExamplePackage.greet()

.. code-block:: julia
    :class: no-execute

    foo() # exported, so don't need to qualify the namespace

**Note:** If you didn't follow the instructions to add a :ref:`startup file <jl_startup_file>`, you may need to quit your REPL and load the package again

Jupyter Workflow
------------------------

We can also work with the package from a Jupyter notebook

Let's create a new output directory in our project, and run ``jupyter lab`` from it

Create a new notebook ``output.ipynb``

.. figure:: /_static/figures/testing-output.png
    :width: 100%

From here, we can use our package's functions as we would functions from other packages

This lets us produce neat output documents, without pasting the whole codebase

We can also run package operations inside the notebook

.. figure:: /_static/figures/testing-notebook.png
    :width: 100%

The change will be reflected in the ``Project.toml`` file

Note that, as usual, we had to first activate ``ExamplePackage`` first before making our dependency changes

.. code-block:: julia
    :class: no-execute

    name = "ExamplePackage"
    uuid = "f85830d0-e1f0-11e8-2fad-8762162ab251"
    authors = ["QuantEcon User <quanteconuser@gmail.com>"]
    version = "0.1.0"

    [deps]
    Distributions = "31c24e10-a181-5473-b8eb-7969acd0382f"
    Expectations = "2fe49d83-0758-5602-8f54-1f90ad0d522b"
    Parameters = "d96e819e-fc66-5662-9728-84c9c7592b0a"

    [extras]
    Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

    [targets]
    test = ["Test"]

There will also be changes in the Manifest as well 

Be sure to add ``output/.ipynb_checkpoints`` to your ``.gitignore`` file, so that's not checked in

Make sure you've activated the project environment (``] activate ExamplePackage``) before you try to propagate changes

Collaborative Work
--------------------------

For someone else to get the package, they simply need to

* Run the following command

.. code-block:: julia
    :class: no-execute

    ] dev https://github.com/quanteconuser/ExamplePackage.jl.git

This will place the repository inside their ``~/.julia/dev`` folder

* Drag-and-drop the folder to GitHub desktop in the usual way

Recall that the path to your ``~/.julia`` folder is

.. code-block:: julia

    DEPOT_PATH[1]

They can then collaborate as they would on other git repositories

In particular, they can run

.. code-block:: julia
    :class: no-execute

    ] activate ExamplePackage

to load the list of dependencies, and

.. code-block:: julia
    :class: no-execute

    ] instantiate

to make sure they are installed on the local machine

Unit Testing
====================================

It's important to make sure that your code is well-tested

There are a few different kinds of test, each with different purposes

* *Unit testing* makes sure that individual pieces of a project work as expected

* *Integration testing* makes sure that they fit together as expected

* *Regression testing* makes sure that behavior is unchanged over time

In this lecture, we'll focus on unit testing

In general, well-written unit tests (which also guard against regression, for example by comparing function output to hardcoded values) are sufficient for most small projects

The ``Test`` Module
-------------------------

Julia provides testing features through a built-in package called ``Test``, which we get by ``using Test``

The basic object is the macro ``@test``

.. code-block:: julia

    using Test
    @test 1 == 1
    @test 1 ≈ 1

Tests will pass if the condition is ``true``, or fail otherwise

If a test is failing, we should flag it with ``@test_broken`` as below

.. code-block:: julia

    @test_broken 1 == 2

This way, we still have access to information about the test, instead of just deleting it or commenting it out

There are other test macros, that check for things like error handling and type-stability

Advanced users can check the `Julia docs <https://docs.julialang.org/en/v1/stdlib/Test/>`_

Example
-----------

Let's add some unit tests for the ``foo()`` function we defined earlier

Our ``tests/runtests.jl`` file should look like this

**As before, this should be pasted into the file directly**

.. code-block:: julia
    :class: no-execute

    using ExamplePackage
    using Test

    @test foo() ≈ 0.11388071406436832
    @test foo(1, 1.5) ≈ 0.2731856314283442
    @test_broken foo(1, 0) # tells us this is broken

And run it by typing ``] test`` into an activated REPL (i.e., a REPL where you've run ``] activate ExamplePackage``)

Test Sets
-------------

By default, the ``runtests.jl`` folder starts off with a ``@testset``

This is useful for organizing different batches of tests, but for now we can simply ignore it

To learn more about test sets, see `the docs <https://docs.julialang.org/en/v1/stdlib/Test/index.html#Working-with-Test-Sets-1/>`_

Running Tests
-----------------------

There are a few different ways to run the tests for your package

* Run the actual ``runtests.jl``, say by hitting ``shift-enter`` on it in Atom

* From a fresh (``v1.1``) REPL, run ``] test ExamplePackage``

* From an activated (``ExamplePackage``) REPL, simply run ``] test`` (recall that you can activate with ``] activate ExamplePackage``)

Continuous Integration with Travis
==========================================

Setup
-------

By default, Travis should have access to all your repositories and deploy automatically

This includes private repos if you're on a student developer pack or an academic plan (Travis detects this automatically)

To change this, go to "settings" under your GitHub profile

.. figure:: /_static/figures/git-settings.png
    :width: 50%

Click "Applications," then "Travis CI," then "Configure," and choose the repos you want to be tracked

Build Options
----------------

By default, Travis will compile and test your project (i.e., "build" it) for new commits and PRs for every tracked repo with a ``.travis.yml`` file

We can see ours by opening it in Atom

.. code-block:: julia
    :class: no-execute

    # Documentation: http://docs.travis-ci.com/user/languages/julia/
    language: julia
    os:
    - linux
    - osx
    julia:
    - 1.1
    - nightly
    matrix:
    allow_failures:
        - julia: nightly
    fast_finish: true
    notifications:
    email: false
    after_success:
    - julia -e 'using Pkg; Pkg.add("Coverage"); using Coverage; Codecov.submit(process_folder())'

This is telling Travis to build the project in Julia, on OSX and Linux, using Julia v1.1 and the latest development build ("nightly")

It also says that if the nightly version doesn't work, that shouldn't register as a failure

**Note** You won't need OSX unless you're building something Mac-specific, like iOS or Swift

You can delete those lines to speed up the build, likewise for the nightly Julia version

Working with Builds
--------------------

As above, builds are triggered whenever we push changes or open a pull request

For example, if we push our changes to the server and then click the Travis badge (the one which says "build") on the README, we should see something like

.. figure:: /_static/figures/travis-progress.png
    :width: 100%

Note that you may need to wait a bit and/or refresh your browser

This gives us an overview of all the builds running for that commit

To inspect a build more closely (say, if it fails), we can click on it and expand the log options

.. figure:: /_static/figures/travis-log.png
    :width: 100%

Note that the build times here aren't informative, because we can't generally control the hardware to which our job is allocated

We can also cancel specific jobs, either from their specific pages or by clicking the grey "x" button on the dashboard

Lastly, we can trigger builds manually (without a new commit or PR) from the Travis overview

.. figure:: /_static/figures/travis-trigger.png
    :width: 100%

To commit *without* triggering a build, simply add "[ci skip]" somewhere inside the commit message

Travis and Pull Requests
----------------------------

One key feature of Travis is the ability to see at-a-glance whether PRs pass tests before merging them

This happens automatically when Travis is enabled on a repository

For an example of this feature, see `this PR <https://github.com/QuantEcon/Games.jl/pull/65/>`_ in the ``Games.jl`` repository

Code Coverage
===================

Beyond the success or failure of our test suite, we also want to know how much of our code the tests cover

The tool we use to do this is called `Codecov <http://codecov.io>`_

Setup
---------

You'll find that Codecov is automatically enabled for public repos with Travis

For private ones, you'll need to first get an access token

Add private scope in the Codecov website, just like we did for Travis

Navigate to the repo settings page (i.e., ``https://codecov.io/gh/quanteconuser/ExamplePackage.jl/settings`` for our repo) and copy the token

Next, go to your Travis settings and add an environment variable as below

.. figure:: /_static/figures/travis-settings.png
    :width: 100%

Interpreting Results
------------------------

Click the Codecov badge to see the build page for your project

This shows us that our tests cover 50\% of our functions in ``src//``

**Note:** To get a more detailed view, we can click the ``src//`` and the resultant filename

**Note:** Codecov may take a few minutes to run for the first time

.. figure:: /_static/figures/codecov.png
    :width: 100%

This shows us precisely which methods (and parts of methods) are untested

Pull Requests to External Julia Projects
===============================================

As mentioned in :doc:`version control <../more_julia/version_control>`, sometimes we'll want to work on external repos that are also Julia projects

* ``] dev`` the git URL (or package name, if the project is a registered Julia package), which will both clone the git repo to ``~/.julia/dev`` and sync it with the Julia package manager

For example, running

.. code-block:: julia
    :class: no-execute

    ] dev Expectations

will clone the repo ``https://github.com/quantecon/Expectations.jl`` to ``~/.julia/dev/Expectations``

Make sure you do this from the base Julia environment (i.e., after running ``] activate`` without arguments)

As a reminder, you can find the location of your ``~/.julia`` folder (called the "user depot"), by running

.. code-block:: julia

    DEPOT_PATH[1]

The ``] dev`` command will also add the target to the package manager, so that whenever we run ``using Expectations``, Julia will load our cloned copy from that location

.. code-block:: julia
    :class: no-execute

    using Expectations
    pathof(Expectations) # points to our git clone

* Drag that folder to GitHub Desktop

* The next step is to fork the original (external) package from its website (i.e., ``https://github.com/quantecon/Expectations.jl``) to your account (``https://github.com/quanteconuser/Expectations.jl`` in our case)

    .. figure:: /_static/figures/testing-fork.png
        :width: 100%

* Edit the settings in GitHub Desktop (from the "Repository" dropdown) to reflect the new URL

    .. figure:: /_static/figures/testing-repo-settings.png
        :width: 75%

    Here, we'd change the highlighted text to read ``quanteconuser``, or whatever our GitHub ID is

* If you make some changes in a text editor and return to GitHub Desktop, you'll see something like

**Note:** As before, we're editing the files directly in ``~/.julia/dev``, as opposed to cloning the repo again.

.. figure:: /_static/figures/testing-commit.png
    :width: 100%

Here, for example, we're revising the README

* Clicking "commit to master" (recall that the checkboxes next to each file indicate whether it's to be committed) and then pushing (e.g., hitting "push" under the "Repository" dropdown) will add the committed changes to your account

To confirm this, we can check the history on our account `here <https://github.com/quanteconuser/Expectations.jl/commits/master>`__; for more on working with git repositories, see the :doc:`version control <../more_julia/version_control>` lecture

.. figure:: /_static/figures/testing-expectations.png
    :width: 100%

The green check mark indicates that Travis tests passed for this commit

* Clicking "new pull request" from the pull requests tab will show us a snapshot of the changes, and let us create a pull request for project maintainers to review and approve

.. figure:: /_static/figures/testing-pr2.png
    :width: 100%

For more on PRs, see the relevant section of the :doc:`version control <../more_julia/version_control>` lecture

For more on forking, see the docs on `GitHub Desktop <https://help.github.com/desktop/guides/contributing-to-projects/cloning-a-repository-from-github-to-github-desktop/>`_ and `the GitHub Website <https://guides.github.com/activities/forking/>`_


Case with Write Access
---------------------------

If you have write access to the repo, we can skip the preceding steps about forking and changing the URL

You can use ``] dev`` on a package name or the URL of the package.

.. code-block:: julia
    :class: no-execute

    ] dev Expectations

or ``] dev https://github.com/quanteconuser/Expectations.jl.git`` as an example for an unreleased package by URL

Which will again clone the repo to ``~/.julia/dev``, and use it as a Julia package

.. code-block:: julia
    :class: no-execute

    using Expectations
    pathof(Expectations) # points to our git clone

Next, drag that folder to GitHub Desktop as before

Then, in order to work with the package locally, all we need to do is open the ``~/.julia/dev/Expectations`` in a text editor (like Atom)

.. figure:: /_static/figures/testing-atom-package.png
    :width: 100%

From here, we can edit this package just like we created it ourselves and use GitHub Desktop to track versions of our package files (say, after ``] up``, or editing source code, ``] add Package``, etc.)

Removing a Julia Package
------------------------------

To "un-dev" a Julia package (say, if we want to use our old ``Expectations.jl``), you can simply run

.. code-block:: julia
    :class: no-execute

    ] free Expectations

To delete it entirely, simply run

.. code-block:: julia
    :class: no-execute

    ] rm Expectations

From a REPL where that package is in the active environment

Benchmarking
==================

Another goal of testing is to make sure that code doesn't slow down significantly from one version to the next

We can do this using tools provided by the ``BenchmarkTools.jl`` package

See the :doc:`need for speed <../more_julia/need_for_speed>` lecture for more details

Additional Notes
=======================

* The `JuliaCI <https://github.com/JuliaCI/>`_ organization provides more Julia utilities for continuous integration and testing

Review
=============

To review the workflow for creating, versioning, and testing a new project end-to-end

1. Create the local package directory using the ``PkgTemplates.jl``

2. Add that package to the Julia package manager, by opening a Julia REPL in the ``~/.julia/dev/ExamplePackage.jl``, making sure the active environment is the default one ``(v1.1)``, and hitting ``] dev .``

3. Drag-and-drop that folder to GitHub Desktop

4. Create an empty repository with the same name on the GitHub server

5. Push from GitHub Desktop to the server

6. Open **the original project folder** (e.g., ``~/.julia/dev/ExamplePackage.jl``) in Atom

7. Make changes, test, iterate on it, etc. As a rule, functions like should live in the ``src/`` directory once they're stable, and you should export them from that file with ``export func1, func2``. This will export all methods of ``func1``, ``func2``, etc.

8. Commit them in GitHub Desktop as you go (i.e., you can and should use version control to track intermediate states)

9. Push to the server, and see the Travis and Codecov results (note that these may take a few minutes the first time)

Exercises
============

Exercise 1
---------------

Following the :ref:`instructions for a new project <project_setup>`, create a new package on your github account called ``NewtonsMethod.jl``

In this package, you should create a simple package to do Newton's Method using the code you did in the
:ref:`Newton's method <jbe_ex8a>` exercise in :doc:`Introductory Examples <../getting_started_julia/julia_by_example>`

In particular, within your package you should have two functions

* ``newtonroot(f, f′; x₀, tol = 1E-7, maxiter = 1000)``
* ``newtonroot(f; x₀, tol = 1E-7, maxiter = 1000)``

Where the second function uses Automatic Differentiation to call the first.

The package should include

* implementations of those functions in the ``/src`` directory
* comprehensive set of tests
* project and manifest files to replicate your development environment
* automated running of the tests with Travis CI in GitHub

For the tests, you should have at the very minimum

* a way to handle non-convergence (e.g. return back ``nothing`` as discussed in :ref:`error handling <error_handling>`
* several ``@test`` for the root of a known function, given the ``f`` and analytical ``f'`` derivatives
* tests of those roots using the automatic differentiation version of the function
* test of finding those roots with a ``BigFloat`` and not just a ``Float64``
* test of non-convergence for a function without a root (e.g. :math:`f(x) = 2 + x^2` )
* test to ensure that the ``maxiter`` is working (e.g. what happens if you call ``maxiter = 5``
* test to ensure that ``tol`` is working

And anything else you can think of.  You should be able to run ``] test`` for the project to check that the test-suite is running, and then ensure that it is running automatically on Travis CI

Push a commit to the repository which breaks one of the tests and see what the Travis CI reports after running the build
