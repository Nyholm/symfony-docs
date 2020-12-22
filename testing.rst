.. index::
   single: Tests

Testing
=======

Whenever you write a new line of code, you also potentially add new bugs.
To build better and more reliable applications, you should test your code
using both functional and unit tests.

The PHPUnit Testing Framework
-----------------------------

Symfony integrates with an independent library called `PHPUnit`_ to give you a
rich testing framework. This article won't cover PHPUnit itself, which has its
own excellent `documentation`_.

Before creating your first test, install the `PHPUnit Bridge component`_, which
wraps the original PHPUnit binary to provide additional features:

.. code-block:: terminal

    $ composer require --dev symfony/phpunit-bridge

After the library downloads, try executing PHPUnit by running (the first time
you run this, it will download PHPUnit itself and make its classes available in
your app):

.. code-block:: terminal

    $ ./bin/phpunit

.. note::

    The ``./bin/phpunit`` command is created by :ref:`Symfony Flex <symfony-flex>`
    when installing the ``phpunit-bridge`` package. If the command is missing, you
    can remove the package (``composer remove symfony/phpunit-bridge``) and install
    it again. Another solution is to remove the project's ``symfony.lock`` file and
    run ``composer install`` to force the execution of all Symfony Flex recipes.

Each test is a PHP class that should live in the ``tests/`` directory of
your application. If you follow this rule, then you can run all of your
application's tests with the same command as before.

PHPUnit is configured by the ``phpunit.xml.dist`` file in the root of your
Symfony application.

.. tip::

    Use the ``--coverage-*`` command options to generate code coverage reports.
    Read the PHPUnit manual to learn more about `code coverage analysis`_.

Types of Tests
--------------

`Unit Tests`_
    These tests ensure that *individual* units of source code (e.g. a single
    class) behave as intended.

`Integration Tests`_
    These tests test a combination of classes and commonly interact with
    Symfony's service container. These tests do not yet cover the full
    working application, those are called *Functional tests*.

`Functional Tests`_
    Functional tests test the behavior of a complete application. They
    make HTTP requests and test that the response is as expected.

`End to End Tests (E2E)`_
    At last, end to end tests test the application as a real user. They use
    a real browser and real integrations with external services.

Unit Tests
----------

A `unit test`_ ensures that individual units of source code (e.g. a single
class or some specific method in some class) meet their design and behave
as intended. Writing Symfony unit tests is no different from writing
standard PHPUnit unit tests. You can learn about it in the PHPUnit
documentation: `Writing Tests for PHPUnit`_.

By convention, the ``tests/`` directory should replicate the directory
of your application for unit tests. So, if you're testing a class in the
``src/Util/`` directory, put the test in the ``tests/Util/`` directory.
Autoloading is automatically enabled via the ``vendor/autoload.php`` file
(as configured by default in the ``phpunit.xml.dist`` file).

You can run tests using the ``bin/phpunit`` command:

.. code-block:: terminal

    # run all tests of the application
    $ php bin/phpunit

    # run all tests in the Util/ directory
    $ php bin/phpunit tests/Util

    # run tests for the Calculator class
    $ php bin/phpunit tests/Util/CalculatorTest.php

Integration Tests
-----------------

TODO: KernelTestCase

Accessing the Container
~~~~~~~~~~~~~~~~~~~~~~~

You can get the same container used in the application, which only includes
the public services::

    public function testSomething()
    {
        $kernel = self::bootKernel();
        $container = $kernel->getContainer();
        $someService = $container->get('the-service-ID');

        // ...
    }

Symfony tests also have access to a special container that includes both the
public services and the non-removed :ref:`private services <container-public>`
services::

    public function testSomething()
    {
        // this call is needed; otherwise the container will be empty
        self::bootKernel();

        $container = self::$container;
        // $someService = $container->get('the-service-ID');

        // ...
    }

.. TODO is this really different from self::$container and how to access
   this in KernelTestCase?

    Finally, for the most rare edge-cases, Symfony includes a special container
    which provides access to all services, public and private. This special
    container is a service that can be get via the normal container::

        public function testSomething()
        {
            $client = self::createClient();
            $normalContainer = $client->getContainer();
            $specialContainer = $normalContainer->get('test.service_container');

            // $somePrivateService = $specialContainer->get('the-service-ID');

            // ...
        }

Mocking Services
~~~~~~~~~~~~~~~~

TODO

.. _functional-tests:

Functional Tests
----------------

Functional tests check the integration of the different layers of an
application (from the routing to the views). They are no different from unit
tests as far as PHPUnit is concerned, but they have a very specific workflow:

* Make a request;
* Click on a link or submit a form;
* Test the response;
* Rinse and repeat.

Before creating your first test, install the ``symfony/test-pack`` which
requires multiple packages providing some of the utilities used in the
tests:

.. code-block:: terminal

    $ composer require --dev symfony/test-pack

Set-up your Test Environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Client used by functional tests creates a Kernel that runs in a special
``test`` environment. Since Symfony loads the ``config/packages/test/*.yaml``
in the ``test`` environment, you can tweak any of your application's settings
specifically for testing.

For example, by default, the Swift Mailer is configured to *not* actually
deliver emails in the ``test`` environment. You can see this under the ``swiftmailer``
configuration option:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/test/swiftmailer.yaml

        # ...
        swiftmailer:
            disable_delivery: true

    .. code-block:: xml

        <!-- config/packages/test/swiftmailer.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:swiftmailer="http://symfony.com/schema/dic/swiftmailer"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/swiftmailer
                https://symfony.com/schema/dic/swiftmailer/swiftmailer-1.0.xsd">

            <!-- ... -->
            <swiftmailer:config disable-delivery="true"/>
        </container>

    .. code-block:: php

        // config/packages/test/swiftmailer.php

        // ...
        $container->loadFromExtension('swiftmailer', [
            'disable_delivery' => true,
        ]);

You can also use a different environment entirely, or override the default
debug mode (``true``) by passing each as options to the ``createClient()``
method::

    $client = static::createClient([
        'environment' => 'my_test_env',
        'debug'       => false,
    ]);

.. tip::

    It is recommended to run your test with ``debug`` set to ``false`` on
    your CI server, as it significantly improves test performance.

Customizing Environment Variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to customize some environment variables for your tests (e.g. the
``DATABASE_URL`` used by Doctrine), you can do that by overriding anything you
need in your ``.env.test`` file:

.. code-block:: text

    # .env.test
    DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name_test?serverVersion=5.7"

    # use SQLITE
    # DATABASE_URL="sqlite:///%kernel.project_dir%/var/app.db"

This file is automatically read in the ``test`` environment: any keys here override
the defaults in ``.env``.

Configuring a Database for Tests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tests that interact with the database should use their own separate database to
not mess with the databases used in the other :ref:`configuration environments <configuration-environments>`.
To do that, edit or create the ``.env.test.local`` file at the root directory of
your project and define the new value for the ``DATABASE_URL`` env var:

.. code-block:: bash

    # .env.test.local
    DATABASE_URL="mysql://USERNAME:PASSWORD@127.0.0.1:3306/DB_NAME?serverVersion=5.7"

.. tip::

    A common practice is to append the ``_test`` suffix to the original database
    names in tests. If the database name in production is called ``project_acme``
    the name of the testing database could be ``project_acme_test``.

The above assumes that each developer/machine uses a different database for the
tests. If the entire team uses the same settings for tests, edit or create the
``.env.test`` file instead and commit it to the shared repository. Learn more
about :ref:`using multiple .env files in Symfony applications <configuration-multiple-env-files>`.

Resetting the Database Automatically Before each Test
.....................................................

Tests should be independent from each other to avoid side effects. For example,
if some test modifies the database (by adding or removing an entity) it could
change the results of other tests. Run the following command to install a bundle
that ensures that each test is run with the same unmodified database:

.. code-block:: terminal

    $ composer require --dev dama/doctrine-test-bundle

Now, enable it as a PHPUnit extension or listener:

.. code-block:: xml

    <!-- phpunit.xml.dist -->
    <phpunit>
        <!-- ... -->

        <extensions>
            <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
        </extensions>
    </phpunit>

This bundle uses a clever trick to avoid side effects without sacrificing
performance: it begins a database transaction before every test and rolls it
back automatically after the test finishes to undo all changes. Read more in the
documentation of the `DAMADoctrineTestBundle`_.

.. _doctrine-fixtures:

Load Dummy Data Fixtures
........................

Instead of using the real data from the production database, it's common to use
fake or dummy data in the test database. This is usually called *"fixtures data"*
and Doctrine provides a library to create and load them. Install it with:

.. code-block:: terminal

    $ composer require --dev doctrine/doctrine-fixtures-bundle

Then, use the ``make:fixtures`` command to generate an empty fixture class:

.. code-block:: terminal

    $ php bin/console make:fixtures

    The class name of the fixtures to create (e.g. AppFixtures):
    > ProductFixture

Customize the new class to load ``Product`` objects into Doctrine::

    // src/DataFixtures/ProductFixture.php
    namespace App\DataFixtures;

    use App\Entity\Product;
    use Doctrine\Bundle\FixturesBundle\Fixture;
    use Doctrine\Persistence\ObjectManager;

    class ProductFixture extends Fixture
    {
        public function load(ObjectManager $manager)
        {
            $product = new Product();
            $product->setName('Priceless widget');
            $product->setPrice(14.50);
            $product->setDescription('Ok, I guess it *does* have a price');
            $manager->persist($product);

            // add more products

            $manager->flush();
        }
    }

Empty the database and reload *all* the fixture classes with:

.. code-block:: terminal

    $ php bin/console doctrine:fixtures:load

For more information, read the `DoctrineFixturesBundle documentation`_.

Write Your First Functional Test
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Functional tests are PHP files that typically live in the ``tests/Controller``
directory of your application. If you want to test the pages handled by your
``PostController`` class, start by creating a new ``PostControllerTest.php``
file that extends a special ``WebTestCase`` class.

As an example, a test could look like this::

    // tests/Controller/PostControllerTest.php
    namespace App\Tests\Controller;

    use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

    class PostControllerTest extends WebTestCase
    {
        public function testShowPost()
        {
            $client = static::createClient();

            $client->request('GET', '/post/hello-world');

            $this->assertEquals(200, $client->getResponse()->getStatusCode());
        }
    }

.. tip::

    To run your functional tests, the ``WebTestCase`` class needs to know which
    is the application kernel to bootstrap it. The kernel class is usually
    defined in the ``KERNEL_CLASS`` environment variable (included in the
    default ``.env.test`` file provided by Symfony):

    If your use case is more complex, you can also override the
    ``createKernel()`` or ``getKernelClass()`` methods of your functional test,
    which take precedence over the ``KERNEL_CLASS`` env var.

In the above example, you validated that the HTTP response was successful. The
next step is to validate that the page actually contains the expected content.
The ``createClient()`` method returns a client, which is like a browser that
you'll use to crawl your site::

    $crawler = $client->request('GET', '/post/hello-world');

The ``request()`` method (read
:ref:`more about the request method <testing-request-method-sidebar>`)
returns a :class:`Symfony\\Component\\DomCrawler\\Crawler` object which can
be used to select elements in the response, click on links and submit forms.

Useful Assertions
~~~~~~~~~~~~~~~~~

To get you started faster, here is a list of the most common and
useful test assertions::

    use Symfony\Component\HttpFoundation\Response;

    // ...

    // asserts that there is at least one h2 tag with the class "subtitle"
    // the third argument is an optional message shown on failed tests
    $this->assertGreaterThan(0, $crawler->filter('h2.subtitle')->count(),
        'There is at least one subtitle'
    );

    // asserts that there are exactly 4 h2 tags on the page
    $this->assertCount(4, $crawler->filter('h2'));

    // asserts that the "Content-Type" header is "application/json"
    $this->assertResponseHeaderSame('Content-Type', 'application/json');
    // equivalent to:
    $this->assertTrue($client->getResponse()->headers->contains(
        'Content-Type', 'application/json'
    ));

    // asserts that the response content contains a string
    $this->assertContains('foo', $client->getResponse()->getContent());
    // ...or matches a regex
    $this->assertRegExp('/foo(bar)?/', $client->getResponse()->getContent());

    // asserts that the response status code is 2xx
    $this->assertResponseIsSuccessful();
    // equivalent to:
    $this->assertTrue($client->getResponse()->isSuccessful());

    // asserts that the response status code is 404 Not Found
    $this->assertTrue($client->getResponse()->isNotFound());

    // asserts a specific status code
    $this->assertResponseStatusCodeSame(201);
    // HTTP status numbers are available as constants too:
    // e.g. 201 === Symfony\Component\HttpFoundation\Response::HTTP_CREATED
    // equivalent to:
    $this->assertEquals(201, $client->getResponse()->getStatusCode());

    // asserts that the response is a redirect to /demo/contact
    $this->assertResponseRedirects('/demo/contact');
    // equivalent to:
    $this->assertTrue($client->getResponse()->isRedirect('/demo/contact'));
    // ...or check that the response is a redirect to any URL
    $this->assertResponseRedirects();

.. versionadded:: 4.3

    The ``assertResponseHeaderSame()``, ``assertResponseIsSuccessful()``,
    ``assertResponseStatusCodeSame()``, ``assertResponseRedirects()`` and other
    related methods were introduced in Symfony 4.3.

Working with the Test Client
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The test client simulates an HTTP client like a browser and makes requests
into your Symfony application::

    $crawler = $client->request('GET', '/post/hello-world');

The ``request()`` method takes the HTTP method and a URL as arguments and
returns a ``Crawler`` instance.

.. tip::

    Hardcoding the request URLs is a best practice for functional tests. If the
    test generates URLs using the Symfony router, it won't detect any change
    made to the application URLs which may impact the end users.

.. _testing-request-method-sidebar:

.. sidebar:: More about the ``request()`` Method:

    The full signature of the ``request()`` method is::

        request(
            $method,
            $uri,
            array $parameters = [],
            array $files = [],
            array $server = [],
            $content = null,
            $changeHistory = true
        )

    The ``server`` array is the raw values that you'd expect to normally
    find in the PHP `$_SERVER`_ superglobal. For example, to set the
    ``Content-Type`` and ``Referer`` HTTP headers, you'd pass the following (mind
    the ``HTTP_`` prefix for non standard headers)::

        $client->request(
            'GET',
            '/post/hello-world',
            [],
            [],
            [
                'CONTENT_TYPE' => 'application/json',
                'HTTP_REFERER' => '/foo/bar',
            ]
        );

Use the crawler to find DOM elements in the response. These elements can then
be used to click on links and submit forms::

    $crawler = $client->clickLink('Go elsewhere...');

    $crawler = $client->submitForm('validate', ['name' => 'Fabien']);

The ``clickLink()`` and ``submitForm()`` methods both return a ``Crawler`` object.
These methods are the best way to browse your application as it takes care
of a lot of things for you, like detecting the HTTP method from a form and
giving you a nice API for uploading files.

The ``request()`` method can also be used to simulate form submissions directly
or perform more complex requests. Some useful examples::

    // submits a form directly (but using the Crawler is easier!)
    $client->request('POST', '/submit', ['name' => 'Fabien']);

    // submits a raw JSON string in the request body
    $client->request(
        'POST',
        '/submit',
        [],
        [],
        ['CONTENT_TYPE' => 'application/json'],
        '{"name":"Fabien"}'
    );

    // Form submission with a file upload
    use Symfony\Component\HttpFoundation\File\UploadedFile;

    $photo = new UploadedFile(
        '/path/to/photo.jpg',
        'photo.jpg',
        'image/jpeg',
        null
    );
    $client->request(
        'POST',
        '/submit',
        ['name' => 'Fabien'],
        ['photo' => $photo]
    );

    // Perform a DELETE request and pass HTTP headers
    $client->request(
        'DELETE',
        '/post/12',
        [],
        [],
        ['PHP_AUTH_USER' => 'username', 'PHP_AUTH_PW' => 'pa$$word']
    );

Last but not least, you can force each request to be executed in its own PHP
process to avoid any side effects when working with several clients in the same
script::

    $client->insulate();

AJAX Requests
.............

The Client provides a :method:`Symfony\\Component\\BrowserKit\\AbstractBrowser::xmlHttpRequest`
method, which has the same arguments as the ``request()`` method, and it's a
shortcut to make AJAX requests::

    // the required HTTP_X_REQUESTED_WITH header is added automatically
    $client->xmlHttpRequest('POST', '/submit', ['name' => 'Fabien']);

Browsing
........

The Client supports many operations that can be done in a real browser::

    $client->back();
    $client->forward();
    $client->reload();

    // clears all cookies and the history
    $client->restart();

.. note::

    The ``back()`` and ``forward()`` methods skip the redirects that may have
    occurred when requesting a URL, as normal browsers do.

Accessing Internal Objects
..........................

If you use the client to test your application, you might want to access the
client's internal objects::

    $history = $client->getHistory();
    $cookieJar = $client->getCookieJar();

You can also get the objects related to the latest request::

    // the HttpKernel request instance
    $request = $client->getRequest();

    // the BrowserKit request instance
    $request = $client->getInternalRequest();

    // the HttpKernel response instance
    $response = $client->getResponse();

    // the BrowserKit response instance
    $response = $client->getInternalResponse();

    // the Crawler instance
    $crawler = $client->getCrawler();

Accessing the Profiler Data
...........................

On each request, you can enable the Symfony profiler to collect data about the
internal handling of that request. For example, the profiler could be used to
verify that a given page runs less than a certain number of database
queries when loading.

To get the Profiler for the last request, do the following::

    // enables the profiler for the very next request
    $client->enableProfiler();

    $crawler = $client->request('GET', '/profiler');

    // gets the profile
    $profile = $client->getProfile();

For specific details on using the profiler inside a test, see the
:doc:`/testing/profiling` article.

Redirecting
...........

When a request returns a redirect response, the client does not follow
it automatically. You can examine the response and force a redirection
afterwards with the ``followRedirect()`` method::

    $crawler = $client->followRedirect();

If you want the client to automatically follow all redirects, you can
force them by calling the ``followRedirects()`` method before performing the request::

    $client->followRedirects();

If you pass ``false`` to the ``followRedirects()`` method, the redirects
will no longer be followed::

    $client->followRedirects(false);

Sending Custom Headers
......................

If your application behaves according to some HTTP headers, pass them as the
second argument of ``createClient()``::

    $client = static::createClient([], [
        'HTTP_HOST'       => 'en.example.com',
        'HTTP_USER_AGENT' => 'MySuperBrowser/1.0',
    ]);

You can also override HTTP headers on a per request basis::

    $client->request('GET', '/', [], [], [
        'HTTP_HOST'       => 'en.example.com',
        'HTTP_USER_AGENT' => 'MySuperBrowser/1.0',
    ]);

.. tip::

    The test client is available as a service in the container in the ``test``
    environment (or wherever the :ref:`framework.test <reference-framework-test>`
    option is enabled). This means you can override the service entirely
    if you need to.

Reporting Exceptions
....................

Debugging exceptions in functional tests may be difficult because by default
they are caught and you need to look at the logs to see which exception was
thrown. Disabling catching of exceptions in the test client allows the exception
to be reported by PHPUnit::

    $client->catchExceptions(false);

.. index::
   single: Tests; Crawler

.. _testing-crawler:

The Crawler
~~~~~~~~~~~

A Crawler instance is returned each time you make a request with the Client.
It allows you to traverse HTML documents, select nodes, find links and forms.

Traversing
..........

Like jQuery, the Crawler has methods to traverse the DOM of an HTML/XML
document. For example, the following finds all ``input[type=submit]`` elements,
selects the last one on the page, and then selects its immediate parent element::

    $newCrawler = $crawler->filter('input[type=submit]')
        ->last()
        ->parents()
        ->first()
    ;

Many other methods are also available:

``filter('h1.title')``
    Nodes that match the CSS selector.
``filterXpath('h1')``
    Nodes that match the XPath expression.
``eq(1)``
    Node for the specified index.
``first()``
    First node.
``last()``
    Last node.
``siblings()``
    Siblings.
``nextAll()``
    All following siblings.
``previousAll()``
    All preceding siblings.
``parents()``
    Returns the parent nodes.
``children()``
    Returns children nodes.
``reduce($lambda)``
    Nodes for which the callable does not return false.

Since each of these methods returns a new ``Crawler`` instance, you can
narrow down your node selection by chaining the method calls::

    $crawler
        ->filter('h1')
        ->reduce(function ($node, $i) {
            if (!$node->attr('class')) {
                return false;
            }
        })
        ->first()
    ;

.. tip::

    Use the ``count()`` function to get the number of nodes stored in a Crawler:
    ``count($crawler)``

Extracting Information
......................

The Crawler can extract information from the nodes::

    // returns the attribute value for the first node
    $crawler->attr('class');

    // returns the node value for the first node
    $crawler->text();

    // returns the default text if the node does not exist
    $crawler->text('Default text content');

    // pass TRUE as the second argument of text() to remove all extra white spaces, including
    // the internal ones (e.g. "  foo\n  bar    baz \n " is returned as "foo bar baz")
    $crawler->text(null, true);

    // extracts an array of attributes for all nodes
    // (_text returns the node value)
    // returns an array for each element in crawler,
    // each with the value and href
    $info = $crawler->extract(['_text', 'href']);

    // executes a lambda for each node and return an array of results
    $data = $crawler->each(function ($node, $i) {
        return $node->attr('href');
    });

.. versionadded:: 4.4

    The option to trim white spaces in ``text()`` was introduced in Symfony 4.4.

Links
.....

Use the ``clickLink()`` method to click on the first link that contains the
given text (or the first clickable image with that ``alt`` attribute)::

    $client = static::createClient();
    $client->request('GET', '/post/hello-world');

    $client->clickLink('Click here');

If you need access to the :class:`Symfony\\Component\\DomCrawler\\Link` object
that provides helpful methods specific to links (such as ``getMethod()`` and
``getUri()``), use the ``selectLink()`` method instead::

    $client = static::createClient();
    $crawler = $client->request('GET', '/post/hello-world');

    $link = $crawler->selectLink('Click here')->link();
    $client->click($link);

Forms
.....

Use the ``submitForm()`` method to submit the form that contains the given button::

    $client = static::createClient();
    $client->request('GET', '/post/hello-world');

    $crawler = $client->submitForm('Add comment', [
        'comment_form[content]' => '...',
    ]);

The first argument of ``submitForm()`` is the text content, ``id``, ``value`` or
``name`` of any ``<button>`` or ``<input type="submit">`` included in the form.
The second optional argument is used to override the default form field values.

.. note::

    Notice that you select form buttons and not forms as a form can have several
    buttons; if you use the traversing API, keep in mind that you must look for a
    button.

If you need access to the :class:`Symfony\\Component\\DomCrawler\\Form` object
that provides helpful methods specific to forms (such as ``getUri()``,
``getValues()`` and ``getFields()``) use the ``selectButton()`` method instead::

    $client = static::createClient();
    $crawler = $client->request('GET', '/post/hello-world');

    $buttonCrawlerNode = $crawler->selectButton('submit');

    // select the form that contains this button
    $form = $buttonCrawlerNode->form();

    // you can also pass an array of field values that overrides the default ones
    $form = $buttonCrawlerNode->form([
        'my_form[name]'    => 'Fabien',
        'my_form[subject]' => 'Symfony rocks!',
    ]);

    // you can pass a second argument to override the form HTTP method
    $form = $buttonCrawlerNode->form([], 'DELETE');

    // submit the Form object
    $client->submit($form);

The field values can also be passed as a second argument of the ``submit()``
method::

    $client->submit($form, [
        'my_form[name]'    => 'Fabien',
        'my_form[subject]' => 'Symfony rocks!',
    ]);

For more complex situations, use the ``Form`` instance as an array to set the
value of each field individually::

    // changes the value of a field
    $form['my_form[name]'] = 'Fabien';
    $form['my_form[subject]'] = 'Symfony rocks!';

There is also a nice API to manipulate the values of the fields according to
their type::

    // selects an option or a radio
    $form['country']->select('France');

    // ticks a checkbox
    $form['like_symfony']->tick();

    // uploads a file
    $form['photo']->upload('/path/to/lucas.jpg');

    // In the case of a multiple file upload
    $form['my_form[field][0]']->upload('/path/to/lucas.jpg');
    $form['my_form[field][1]']->upload('/path/to/lisa.jpg');

.. tip::

    Instead of hardcoding the form name as part of the field names (e.g.
    ``my_form[...]`` in previous examples), you can use the
    :method:`Symfony\\Component\\DomCrawler\\Form::getName` method to get the
    form name.

    .. versionadded:: 4.4

        The ``getName()`` method was introduced in Symfony 4.4.

.. tip::

    If you purposefully want to select "invalid" select/radio values, see
    :ref:`components-dom-crawler-invalid`.

.. tip::

    You can get the values that will be submitted by calling the ``getValues()``
    method on the ``Form`` object. The uploaded files are available in a
    separate array returned by ``getFiles()``. The ``getPhpValues()`` and
    ``getPhpFiles()`` methods also return the submitted values, but in the
    PHP format (it converts the keys with square brackets notation - e.g.
    ``my_form[subject]`` - to PHP arrays).

.. tip::

    The ``submit()`` and ``submitForm()`` methods define optional arguments to
    add custom server parameters and HTTP headers when submitting the form::

        $client->submit($form, [], ['HTTP_ACCEPT_LANGUAGE' => 'es']);
        $client->submitForm($button, [], 'POST', ['HTTP_ACCEPT_LANGUAGE' => 'es']);

End to End Tests (E2E)
----------------------

TODO
* panther
* testing javascript
* UX or form collections as example?

PHPUnit Configuration
---------------------

Each application has its own PHPUnit configuration, stored in the
``phpunit.xml.dist`` file. You can edit this file to change the defaults or
create a ``phpunit.xml`` file to set up a configuration for your local machine
only.

.. tip::

    Store the ``phpunit.xml.dist`` file in your code repository and ignore
    the ``phpunit.xml`` file.

By default, only the tests stored in ``tests/`` are run via the ``phpunit`` command,
as configured in the ``phpunit.xml.dist`` file:

.. code-block:: xml

    <!-- phpunit.xml.dist -->
    <phpunit>
        <!-- ... -->
        <testsuites>
            <testsuite name="Project Test Suite">
                <directory>tests</directory>
            </testsuite>
        </testsuites>
        <!-- ... -->
    </phpunit>

But you can add more directories. For instance, the following
configuration adds tests from a custom ``lib/tests`` directory:

.. code-block:: xml

    <!-- phpunit.xml.dist -->
    <phpunit>
        <!-- ... -->
        <testsuites>
            <testsuite name="Project Test Suite">
                <!-- ... --->
                <directory>lib/tests</directory>
            </testsuite>
        </testsuites>
        <!-- ... -->
    </phpunit>

To include other directories in the code coverage, also edit the ``<filter>``
section:

.. code-block:: xml

    <!-- phpunit.xml.dist -->
    <phpunit>
        <!-- ... -->
        <filter>
            <whitelist>
                <!-- ... -->
                <directory>lib</directory>
                <exclude>
                    <!-- ... -->
                    <directory>lib/tests</directory>
                </exclude>
            </whitelist>
        </filter>
        <!-- ... -->
    </phpunit>

Learn more
----------

.. toctree::
    :maxdepth: 1
    :glob:

    testing/*
    /components/dom_crawler
    /components/css_selector

.. _`PHPUnit`: https://phpunit.de/
.. _`documentation`: https://phpunit.readthedocs.io/
.. _`PHPUnit Bridge component`: https://symfony.com/components/PHPUnit%20Bridge
.. _`Writing Tests for PHPUnit`: https://phpunit.readthedocs.io/en/stable/writing-tests-for-phpunit.html
.. _`unit test`: https://en.wikipedia.org/wiki/Unit_testing
.. _`$_SERVER`: https://www.php.net/manual/en/reserved.variables.server.php
.. _`code coverage analysis`: https://phpunit.readthedocs.io/en/9.1/code-coverage-analysis.html
