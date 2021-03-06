.. index::
   single: Security; User provider
   single: Security; Entity provider

How to Load Security Users from the Database (the Entity Provider)
==================================================================

Symfony's security system can load security users from anywhere - like a
database, via Active Directory or an OAuth server. This article will show
you how to load your users from the database via a Doctrine entity.

Introduction
------------

Loading users via a Doctrine entity has 2 basic steps:

#. :ref:`Create your User entity <security-crete-user-entity>`
#. :ref:`Configure security.yaml to load from your entity <security-config-entity-provider>`

Afterwards, you can learn more about :ref:`forbidding inactive users <security-advanced-user-interface>`,
:ref:`using a custom query <authenticating-someone-with-a-custom-entity-provider>`
and :ref:`user serialization to the session <security-serialize-equatable>`

.. _security-crete-user-entity:
.. _the-data-model:

1) Create your User Entity
--------------------------

Before you begin, run this command to add support for the Symfony security:

.. code-block:: terminal

    $ composer require symfony/security-bundle

For this article, suppose that you already have a ``User`` entity
with the following fields: ``id``, ``username``, ``password``,
``email`` and ``isActive``::

    // src/Entity/User.php
    namespace App\Entity;

    use Doctrine\ORM\Mapping as ORM;
    use Symfony\Component\Security\Core\User\UserInterface;

    /**
     * @ORM\Table(name="app_users")
     * @ORM\Entity(repositoryClass="App\Repository\UserRepository")
     */
    class User implements UserInterface, \Serializable
    {
        /**
         * @ORM\Column(type="integer")
         * @ORM\Id
         * @ORM\GeneratedValue(strategy="AUTO")
         */
        private $id;

        /**
         * @ORM\Column(type="string", length=25, unique=true)
         */
        private $username;

        /**
         * @ORM\Column(type="string", length=64)
         */
        private $password;

        /**
         * @ORM\Column(type="string", length=254, unique=true)
         */
        private $email;

        /**
         * @ORM\Column(name="is_active", type="boolean")
         */
        private $isActive;

        public function __construct()
        {
            $this->isActive = true;
            // may not be needed, see section on salt below
            // $this->salt = md5(uniqid('', true));
        }

        public function getUsername()
        {
            return $this->username;
        }

        public function getSalt()
        {
            // you *may* need a real salt depending on your encoder
            // see section on salt below
            return null;
        }

        public function getPassword()
        {
            return $this->password;
        }

        public function getRoles()
        {
            return array('ROLE_USER');
        }

        public function eraseCredentials()
        {
        }

        /** @see \Serializable::serialize() */
        public function serialize()
        {
            return serialize(array(
                $this->id,
                $this->username,
                $this->password,
                // see section on salt below
                // $this->salt,
            ));
        }

        /** @see \Serializable::unserialize() */
        public function unserialize($serialized)
        {
            list (
                $this->id,
                $this->username,
                $this->password,
                // see section on salt below
                // $this->salt
            ) = unserialize($serialized, array('allowed_classes' => false));
        }
    }

To make things shorter, some of the getter and setter methods aren't shown.
But you can generate these manually or with your own IDE.

.. caution::

    In the example above, the User entity's table name is "app_users" because
    "USER" is a SQL reserved word. If you wish to call your table name "user",
    `it must be quoted with backticks`_ to avoid errors. The annotation should
    look like ``@ORM\Table(name="`user`")``.

Next, make sure to :ref:`create the database table <doctrine-creating-the-database-tables-schema>`:

.. code-block:: terminal

    $ php bin/console doctrine:migrations:diff
    $ php bin/console doctrine:migrations:migrate

What's this UserInterface?
~~~~~~~~~~~~~~~~~~~~~~~~~~

So far, this is just a normal entity. But to use this class in the
security system, it must implement
:class:`Symfony\\Component\\Security\\Core\\User\\UserInterface`. This
forces the class to have the five following methods:

* :method:`Symfony\\Component\\Security\\Core\\User\\UserInterface::getRoles`
* :method:`Symfony\\Component\\Security\\Core\\User\\UserInterface::getPassword`
* :method:`Symfony\\Component\\Security\\Core\\User\\UserInterface::getSalt`
* :method:`Symfony\\Component\\Security\\Core\\User\\UserInterface::getUsername`
* :method:`Symfony\\Component\\Security\\Core\\User\\UserInterface::eraseCredentials`

To learn more about each of these, see :class:`Symfony\\Component\\Security\\Core\\User\\UserInterface`.

.. caution::

    The ``eraseCredentials()`` method is only meant to clean up possibly stored
    plain text passwords (or similar credentials). Be careful what to erase
    if your user class is also mapped to a database as the modified object
    will likely be persisted during the request.

What do the serialize and unserialize Methods do?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At the end of each request, the User object is serialized to the session.
On the next request, it's unserialized. To help PHP do this correctly, you
need to implement ``Serializable``. But you don't need to serialize everything:
you only need a few fields (the ones shown above plus a few extra if you added
other important fields to your user entity). On each request, the ``id`` is used
to query for a fresh ``User`` object from the database.

Want to know more? See :ref:`security-serialize-equatable`.

.. _authenticating-someone-against-a-database:
.. _security-config-entity-provider:

2) Configure Security to load from your Entity
----------------------------------------------

Now that you have a ``User`` entity that implements ``UserInterface``, you
just need to tell Symfony's security system about it in ``security.yaml``.

In this example, the user will enter their username and password via HTTP
basic authentication. Symfony will query for a ``User`` entity matching
the username and then check the password (more on passwords in a moment):

.. configuration-block::

    .. code-block:: yaml

        # config/packages/security.yaml
        security:
            encoders:
                App\Entity\User:
                    algorithm: bcrypt

            # ...

            providers:
                our_db_provider:
                    entity:
                        class: App\Entity\User
                        property: username
                        # if you're using multiple entity managers
                        # manager_name: customer

            firewalls:
                main:
                    pattern:    ^/
                    http_basic: ~
                    provider: our_db_provider

            # ...

    .. code-block:: xml

        <!-- config/packages/security.xml -->
        <?xml version="1.0" encoding="UTF-8"?>
        <srv:container xmlns="http://symfony.com/schema/dic/security"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:srv="http://symfony.com/schema/dic/services"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                http://symfony.com/schema/dic/services/services-1.0.xsd">

            <config>
                <encoder class="App\Entity\User" algorithm="bcrypt" />

                <!-- ... -->

                <provider name="our_db_provider">
                    <!-- if you're using multiple entity managers, add:
                         manager-name="customer" -->
                    <entity class="App\Entity\User" property="username" />
                </provider>

                <firewall name="main" pattern="^/" provider="our_db_provider">
                    <http-basic />
                </firewall>

                <!-- ... -->
            </config>
        </srv:container>

    .. code-block:: php

        // config/packages/security.php
        use App\Entity\User;

        $container->loadFromExtension('security', array(
            'encoders' => array(
                User::class => array(
                    'algorithm' => 'bcrypt',
                ),
            ),

            // ...

            'providers' => array(
                'our_db_provider' => array(
                    'entity' => array(
                        'class'    => User::class,
                        'property' => 'username',
                    ),
                ),
            ),
            'firewalls' => array(
                'main' => array(
                    'pattern'    => '^/',
                    'http_basic' => null,
                    'provider'   => 'our_db_provider',
                ),
            ),

            // ...
        ));

First, the ``encoders`` section tells Symfony to expect that the passwords
in the database will be encoded using ``bcrypt``. Second, the ``providers``
section creates a "user provider" called ``our_db_provider`` that knows to
query from your ``App\Entity\User`` entity by the ``username`` property. The
name ``our_db_provider`` isn't important: it just needs to match the value
of the ``provider`` key under your firewall. Or, if you don't set the ``provider``
key under your firewall, the first "user provider" is automatically used.

Creating your First User
~~~~~~~~~~~~~~~~~~~~~~~~

To add users, you can implement a :doc:`registration form </doctrine/registration_form>`
or add some `fixtures`_. This is just a normal entity, so there's nothing
tricky, *except* that you need to encode each user's password. But don't
worry, Symfony gives you a service that will do this for you. See :doc:`/security/password_encoding`
for details.

Below is an export of the ``app_users`` table from MySQL with user ``admin``
and password ``admin`` (which has been encoded).

.. code-block:: terminal

    $ mysql> SELECT * FROM app_users;
    +----+----------+--------------------------------------------------------------+--------------------+-----------+
    | id | username | password                                                     | email              | is_active |
    +----+----------+--------------------------------------------------------------+--------------------+-----------+
    |  1 | admin    | $2a$08$jHZj/wJfcVKlIwr5AvR78euJxYK7Ku5kURNhNx.7.CSIJ3Pq6LEPC | admin@example.com  |         1 |
    +----+----------+--------------------------------------------------------------+--------------------+-----------+

.. sidebar:: Do you need to use a Salt property?

    If you use ``bcrypt`` or ``argon2i``, no. Otherwise, yes. All passwords must
    be hashed with a salt, but ``bcrypt`` and ``argon2i`` do this internally.
    Since this tutorial *does* use ``bcrypt``, the ``getSalt()`` method in
    ``User`` can just return ``null`` (it's not used). If you use a different
    algorithm, you'll need to uncomment the ``salt`` lines in the ``User``
    entity and add a persisted ``salt`` property.

.. _security-advanced-user-interface:

Forbid Inactive Users (AdvancedUserInterface)
---------------------------------------------

.. versionadded:: 4.1
    The ``AdvancedUserInterface`` class was deprecated in Symfony 4.1 and no
    alternative is provided. If you need this functionality in your application,
    implement :doc:`a custom user checker </security/user_checkers>` that
    performs the needed checks.

If a User's ``isActive`` property is set to ``false`` (i.e. ``is_active``
is 0 in the database), the user will still be able to login to the site
normally. This is easily fixable.

To exclude inactive users, change your ``User`` class to implement
:class:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface`.
This extends :class:`Symfony\\Component\\Security\\Core\\User\\UserInterface`,
so you only need the new interface::

    // src/Entity/User.php

    use Symfony\Component\Security\Core\User\AdvancedUserInterface;
    // ...

    class User implements AdvancedUserInterface, \Serializable
    {
        // ...

        public function isAccountNonExpired()
        {
            return true;
        }

        public function isAccountNonLocked()
        {
            return true;
        }

        public function isCredentialsNonExpired()
        {
            return true;
        }

        public function isEnabled()
        {
            return $this->isActive;
        }

        // serialize and unserialize must be updated - see below
        public function serialize()
        {
            return serialize(array(
                // ...
                $this->isActive,
            ));
        }
        public function unserialize($serialized)
        {
            list (
                // ...
                $this->isActive,
            ) = unserialize($serialized);
        }
    }

The :class:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface`
interface adds four extra methods to validate the account status:

* :method:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface::isAccountNonExpired`
  checks whether the user's account has expired;
* :method:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface::isAccountNonLocked`
  checks whether the user is locked;
* :method:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface::isCredentialsNonExpired`
  checks whether the user's credentials (password) has expired;
* :method:`Symfony\\Component\\Security\\Core\\User\\AdvancedUserInterface::isEnabled`
  checks whether the user is enabled.

If *any* of these return ``false``, the user won't be allowed to login. You
can choose to have persisted properties for all of these, or whatever you
need (in this example, only ``isActive`` pulls from the database).

So what's the difference between the methods? Each returns a slightly different
error message (and these can be translated when you render them in your login
template to customize them further).

.. note::

    If you use ``AdvancedUserInterface``, you also need to add any of the
    properties used by these methods (like ``isActive``) to the ``serialize()``
    and ``unserialize()`` methods. If you *don't* do this, your user may
    not be deserialized correctly from the session on each request.

Congrats! Your database-loading security system is all setup! Next, add a
true :doc:`login form </security/form_login_setup>` instead of HTTP Basic
or keep reading for other topics.

.. _authenticating-someone-with-a-custom-entity-provider:

Using a Custom Query to Load the User
-------------------------------------

It would be great if a user could login with their username *or* email, as
both are unique in the database. Unfortunately, the native entity provider
is only able to handle querying via a single property on the user.

To do this, make your ``UserRepository`` implement a special
:class:`Symfony\\Bridge\\Doctrine\\Security\\User\\UserLoaderInterface`. This
interface only requires one method: ``loadUserByUsername($username)``::

    // src/Repository/UserRepository.php
    namespace App\Repository;

    use Symfony\Bridge\Doctrine\Security\User\UserLoaderInterface;
    use Doctrine\ORM\EntityRepository;

    class UserRepository extends EntityRepository implements UserLoaderInterface
    {
        public function loadUserByUsername($username)
        {
            return $this->createQueryBuilder('u')
                ->where('u.username = :username OR u.email = :email')
                ->setParameter('username', $username)
                ->setParameter('email', $username)
                ->getQuery()
                ->getOneOrNullResult();
        }
    }

To finish this, just remove the ``property`` key from the user provider in
``security.yaml``:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/security.yaml
        security:
            # ...

            providers:
                our_db_provider:
                    entity:
                        class: App\Entity\User

    .. code-block:: xml

        <!-- config/packages/security.xml -->
        <?xml version="1.0" encoding="UTF-8"?>
        <srv:container xmlns="http://symfony.com/schema/dic/security"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:srv="http://symfony.com/schema/dic/services"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                http://symfony.com/schema/dic/services/services-1.0.xsd">

            <config>
                <!-- ... -->

                <provider name="our_db_provider">
                    <entity class="App\Entity\User" />
                </provider>
            </config>
        </srv:container>

    .. code-block:: php

        // config/packages/security.php
        use App\Entity\User;

        $container->loadFromExtension('security', array(
            // ...

            'providers' => array(
                'our_db_provider' => array(
                    'entity' => array(
                        'class' => User::class,
                    ),
                ),
            ),
        ));

This tells Symfony to *not* query automatically for the User. Instead, when
someone logs in, the ``loadUserByUsername()`` method on ``UserRepository``
will be called.

.. _security-serialize-equatable:

Understanding serialize and how a User is Saved in the Session
--------------------------------------------------------------

If you're curious about the importance of the ``serialize()`` method inside
the ``User`` class or how the User object is serialized or deserialized, then
this section is for you. If not, feel free to skip this.

Once the user is logged in, the entire User object is serialized into the
session. On the next request, the User object is deserialized. Then, the value
of the ``id`` property is used to re-query for a fresh User object from the
database. Finally, the fresh User object is compared to the deserialized
User object to make sure that they represent the same user. For example, if
the ``username`` on the 2 User objects doesn't match for some reason, then
the user will be logged out for security reasons.

Even though this all happens automatically, there are a few important side-effects.

First, the :phpclass:`Serializable` interface and its ``serialize()`` and ``unserialize()``
methods have been added to allow the ``User`` class to be serialized
to the session. This may or may not be needed depending on your setup,
but it's probably a good idea. In theory, only the ``id`` needs to be serialized,
because the :method:`Symfony\\Bridge\\Doctrine\\Security\\User\\EntityUserProvider::refreshUser`
method refreshes the user on each request by using the ``id`` (as explained
above). This gives us a "fresh" User object.

But Symfony also uses the ``username``, ``salt``, and ``password`` to verify
that the User has not changed between requests (it also calls your ``AdvancedUserInterface``
methods if you implement it). Failing to serialize these may cause you to
be logged out on each request. If your user implements the
:class:`Symfony\\Component\\Security\\Core\\User\\EquatableInterface`,
then instead of these properties being checked, your :method:`Symfony\\Component\\Security\\Core\\User\\EquatableInterface::isEqualTo` method
is called, and you can check whatever properties you want. Unless
you understand this, you probably *won't* need to implement this interface
or worry about it.

.. _fixtures: https://symfony.com/doc/master/bundles/DoctrineFixturesBundle/index.html
.. _FOSUserBundle: https://github.com/FriendsOfSymfony/FOSUserBundle
.. _`it must be quoted with backticks`: http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#quoting-reserved-words
