
.. include:: /Includes.rst.txt
.. index:: !Dependency injection
.. _DependencyInjection:
.. _dependency-injection:

====================
Dependency injection
====================

.. versionadded:: 10.0

   :doc:`Changelog/10.0/Feature-84112-SymfonyDependencyInjectionForCoreAndExtbase`

TYPO3 uses a dependency injection solution based on the corresponding `PSR-11 <https://www.php-fig.org/psr/psr-11/>`_
compliant Symfony component to standardize object initialization throughout the Core as well as in extensions.

The recommended way of injecting dependencies is to use constructor injection::

   public function __construct(Dependency $dependency)
   {
       $this->dependency = $dependency;
   }

By default all classes shipped by the TYPO3 Core  system extensions are available for dependency
injection.

.. contents::
   :depth: 3

.. index::
   pair: Dependency injection; Extensions
   pair: Dependency injection; Services
   File; EXT:{extkey}/Configuration/Services.yaml
.. _configure-dependency-injection-in-extensions:

Configure dependency injection in Extensions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Extensions have to configure their classes to make use of the
dependency injection. This can be done in :file:`Configuration/Services.yaml`.
Alternatively :file:`Configuration/Services.php` can be used.

.. code-block:: yaml

    # Configuration/Services.yaml
    services:
      _defaults:
        autowire: true
        autoconfigure: true
        public: false

      Your\Namespace\:
        resource: '../Classes/*'
        exclude: '../Classes/Domain/Model/*'

This is how a basic :file:`Services.yaml` of an extension looks like. The meaning of :yaml:`autowire`,
:yaml:`autoconfigure` and :yaml:`public` will be explained below.


.. note::

   Whenever service configuration or class dependencies change, the Core cache needs
   to be flushed in the Install Tool to rebuild the compiled Symfony container.
   Flushing all caches from the cache clear menu does not flush the compiled Symfony container.

.. note::

   The path exclusion :yaml:`exclude: '../Classes/Domain/Model/*'` excludes
   your Models from the DI Container, which means you can not inject them or inject
   dependencies into them.
   Models are not Services and should therefore not require dependency injection.
   Also, these Objects are created by the extbase persistence layer which does not support the DI container.

.. _autowire:

Autowire
--------

:yaml:`autowire: true` instructs the dependency injection component
to calculate the required dependencies from type declarations. This works for constructor
and inject methods. The calculation yields to a service initialization recipe
which is cached in php code (in TYPO3 Core  cache).

.. note::

   An extension doesn't need to use autowiring, it is free to manually
   wire dependencies in the service configuration file.

Autoconfigure
-------------

It is suggested to enable :yaml:`autoconfigure: true` as this will automatically
add Symfony service tags based on implemented interfaces or base classes.
For example autoconfiguration ensures that classes which implement
:php:`\TYPO3\CMS\Core\SingletonInterface` will be publicly available from the
Symfony container.


Arguments
---------

In case you turned autowire off or need special arguments set, you can configure
those as well.
This means you could set :yaml:`autowire: false` for an extension but provide the needed
arguments via config specifically for classes you want to.
This can be done in chronological order or by naming them.

.. code-block:: yaml

    # Configuration/Services.yaml
      Vendor\MyExtension\UserFunction\ClassA:
        arguments:
          $argA: '@TYPO3\CMS\Core\Database\ConnectionPool'

      Vendor\MyExtension\UserFunction\ClassB:
        arguments:
          - '@TYPO3\CMS\Core\Database\ConnectionPool'

This enables you to inject concrete objects like the QueryBuilder or Database Connection:

.. code-block:: yaml

   # Configuration/Services.yaml
     querybuilder.pages:
       class: 'TYPO3\CMS\Core\Database\Query\QueryBuilder'
       factory:
         - '@TYPO3\CMS\Core\Database\ConnectionPool'
         - 'getQueryBuilderForTable'
       arguments:
         - 'pages'

     Vendor\MyExtension\UserFunction\ClassA:
       public: true
       arguments:
         - '@querybuilder.pages'

Now you can access the QueryBuilder instance within ClassA. With this you can
call your queries without further instantiation. Be aware to clone your object or
resetting the query parts to prevent side effects in case of multiple usages.

This method of injecting Objects does also work with e.g. extension configurations
or typoscript settings.


Public
------

:yaml:`public: false` is a performance optimization and is therefore suggested to be
set in extensions. This settings controls which services are available
through :php:`\Psr\Container\ContainerInterface->get()`. However some classes, that need to
be public, will be marked public automatically due to :yaml:`autoconfigure: true`.
These classes include Singletons, because they need to be shared with code that uses
:php:`\TYPO3\CMS\Core\Utility\GeneralUtility::makeInstance()` and Extbase controllers.

.. index:: Dependency injection; Public
.. _knowing-what-to-make-public:

Knowing what to make public
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Instances of :php:`\TYPO3\CMS\Core\SingletonInterface` and Extbase controllers are
marked public by default. Additionally some classes cannot be private as well.
As the Symfony documentation puts it:

.. this indent is intentional to create a blockquote!

    "Simply said: A service can be marked as private if you do not want to access it directly from your code."

    -- `Official documentation <https://symfony.com/doc/current/service_container/alias_private.html>`_ for public and private services.

Direct access includes instantiation via :php:`\TYPO3\CMS\Core\Utility\GeneralUtility::makeInstance()`.
This means every class that should make use of dependency injection and is not instantiated via injection
itself but by e.g. :php:`\TYPO3\CMS\Core\Utility\GeneralUtility::makeInstance()` have to be marked
as public. Some examples for this are:

* User Functions
* Non-Extbase Controllers
* Classes registered in Hooks
* Authentication services
* DataProcessors

For such classes an extension can override the global :yaml:`public: false` configuration in the
:file:`Configuration/Services.yaml` for each class.

.. code-block:: yaml

    # Configuration/Services.yaml
    services:
      _defaults:
        autowire: true
        autoconfigure: true
        public: false

      Vendor\MyExtension\:
        resource: '../Classes/*'
        exclude: '../Classes/Domain/Model/*'

      Vendor\MyExtension\UserFunction\ClassA:
        public: true

With this configuration you can use dependency injection in :php:`\Vendor\MyExtension\UserFunction\ClassA`
when it is created in the context of a :ts:`USER` TypoScript object which would not be possible if this
class was private.

.. index:: Dependency injection; Errors
.. _errors-resulting-from-wrong-configuration:

Errors resulting from wrong configuration
-----------------------------------------

When objects using dependency injection are not configured properly, one or more
of the following issues can be the result. In such a case, check whether the
class has to be configured as :yaml:`public: true`.

:php:`ArgumentCountError` is raised on missing dependency injection for
:ref:`constructor-injection`:

.. code-block:: text

   (1/1) ArgumentCountError

   Too few arguments to function Vendor\ExtName\Namespace\Class::__construct(),
   0 passed in typo3/sysext/core/Classes/Utility/GeneralUtility.php on line 3461 and exactly 1 expected

An :php:`Error` is raised on missing dependency injection for
:ref:`method-injection`, once the dependency is used within the code:

.. code-block:: text

   (1/1) Error

   Call to a member function methodName() on null


.. _supported-ways-of-dependency-injection:

Supported ways of dependency injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. deprecated:: 11.4
   The class :php:`\TYPO3\CMS\Extbase\Object\ObjectManager` has been deprecated
   with TYPO3 11.4.  Classes should be updated to avoid both, 
   :php:`\TYPO3\CMS\Extbase\Object\ObjectManager` and
   :php:`\TYPO3\CMS\Core\Utility\GeneralUtility::makeInstance()` whenever possible.

Class dependencies should be injected via constructor injection or
setter methods. Where possible, Symfony  
:ref:`dependency injection<dependency-injection>` should be used for all cases 
where DI is required.


.. index:: Dependency injection; Constructor injection
.. _constructor-injection:

Constructor injection
---------------------

A class dependency can simply be specified as a constructor argument::

   public function __construct(Dependency $dependency)
   {
       $this->dependency = $dependency;
   }


.. index:: Dependency injection; Method injection
.. _method-injection:

Method injection
----------------

As an alternative to constructor injection :php:`injectDependency()` Methods can be used.
Additionally a :php:`setDependency()` will also work if it has the annotation :php:`@required`::


   /**
    * @param MyDependency $myDependency
    */
   public function injectMyDependency(MyDependency $myDependency)
   {
       $this->myDependency = $myDependency;
   }

   /**
    * @param MyOtherDependency $myOtherDependency
    * @required
    */
   public function setMyOtherDependency(MyOtherDependency $myOtherDependency)
   {
       $this->myOtherDependency = $myOtherDependency;
   }


.. index:: Dependency injection; Interface injection
.. _interface-injection:

Interface injection
-------------------

It is possible to inject interfaces as well. If there is only one implementation for a certain
interface the interface injection is simply resolved to this implementation::

   public function __construct(DependencyInterface $dependency)
   {
       $this->dependency = $dependency;
   }

When multiple implementation of the same interface exist, an extension needs to specify which
implementation should be injected when the interface is type hinted. Find out more about how this
is achieved in the official `Symfony documentation <https://symfony.com/doc/current/service_container/autowiring.html#working-with-interfaces>`_.

Dependency injection in a XCLASSed class
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If an existing class (for example an Extbase controller) is extended using XCLASS and additional
dependencies are injected using constructor injection, it must be ensured to add a
reference to the extended class in the :file:`Configuration/Services.yaml` file of the
extending extension as shown in the example below::

   TYPO3\CMS\Belog\Controller\BackendLogController: '@Namespace\Extension\Controller\ExtendedBackendLogController'

Further information
^^^^^^^^^^^^^^^^^^^

* `Symfony dependency injection component <https://symfony.com/doc/current/components/dependency_injection.html>`__
* `Symfony service container <https://symfony.com/doc/current/service_container.html>`_
* :doc:`t3core:Changelog/10.0/Feature-84112-SymfonyDependencyInjectionForCoreAndExtbase`
  of the TYPO3 Core .
