.. index::
   single: Propel

Базы Данных и Propel
====================

Одна из самых распространненных и сложных задач в любом приложении - это 
сохранение информации в базу данных и чтение информации оттуда. Symfony2
не интегрирована с какими-либо ORM, но легко интегрируется с Propel.
Для установки Propel, ознакомьтесь с `Working With Symfony2`_ по поводу 
документации Propel.

Простой пример: Некий продукт
---------------------------

В этом разделе вы настроите свою базу данных, создадите объект ``Product``,
сохраните его в базу данных и извлечете оттуда обратно.

.. sidebar:: Пишем код вместе с примером

    Если вы хотите выполнить пример из этой главы, создайте ``AcmeStoreBundle``
    через:

    .. code-block:: bash

        $ php app/console generate:bundle --namespace=Acme/StoreBundle

Настройка Базы Данных
~~~~~~~~~~~~~~~~~~~~~~~~

Перед тем, как начать, вам нужно конфигурировать информацию о подключении
вашей базы данных. По соглашению, эту информацию обычно настраивают в файле 
``app/config/parameters.yml``:

.. code-block:: yaml

    # app/config/parameters.yml
    parameters:
        database_driver:   mysql
        database_host:     localhost
        database_name:     test_project
        database_user:     root
        database_password: password
        database_charset:  UTF8

.. note::

    Определять конфигурацию через ``parameters.yml`` - это просто условность. На параметры, 
    определенные в этом файле, ссылается основной файл конфигураций при установке Propel:

Эти параметры, определенные в ``parameters.yml`` теперь можно включить в конфигурационный файл
(``config.yml``):

.. code-block:: yaml

    propel:
        dbal:
            driver:     "%database_driver%"
            user:       "%database_user%"
            password:   "%database_password%"
            dsn:        "%database_driver%:host=%database_host%;dbname=%database_name%;charset=%database_charset%"

Теперь, когда Propel знает о вашей базе данных, Symfony2 может создать ее за вас:

.. code-block:: bash

    $ php app/console propel:database:create

.. note::

    В этом примере, у вас одно настроенное соединение, названное ``default``. Если
    вы хотите настроить больше соединений, прочтите раздел `PropelBundle
    configuration section`_.

Создаем Model Class
~~~~~~~~~~~~~~~~~~~~~~

В мире Propel, классы ActiveRecord называются моделями (**models**), потому что классы,
создаваемые Propel, содержат определенную бизнес-логику.

.. note::

    Для тех, кто пользуется Symfony2 и Doctrine2 -  **модели** эквиваленты **сущностям**.

Допустим, вы создаете приложение, где нужно выставлять продукты. Сначала создается
файл``schema.xml`` внутри директории ``Resources/config`` вашего ``AcmeStoreBundle``:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8" ?>
    <database name="default"
        namespace="Acme\StoreBundle\Model"
        defaultIdMethod="native"
    >
        <table name="product">
            <column name="id"
                type="integer"
                required="true"
                primaryKey="true"
                autoIncrement="true"
            />
            <column name="name"
                type="varchar"
                primaryString="true"
                size="100"
            />
            <column name="price"
                type="decimal"
            />
            <column name="description"
                type="longvarchar"
            />
        </table>
    </database>

Построение Модели
~~~~~~~~~~~~~~~~~~

После создания ``schema.xml``, сгенерируйте из него вашу модель, запустив команду:

.. code-block:: bash

    $ php app/console propel:model:build

Таким образом, для быстрой разработки вашего приложения генерируется каждый model class 
в директорию ``Model/`` пакета ``AcmeStoreBundle``.

Создание Таблиц\Схемы Базы Данных 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Теперь у вас есть удобный для использования класс ``Product`` и все, что вам нужно - 
это сохранить его в базу данных. Правда, в вашей базе данных пока отсутствует 
соответсвующая таблицы  ``product``. Но к счастью, Propel автоматически создает все
необходимые таблицы для базы анных для каждой известной модели в вашем приложении. 
Для этого запустите следующее:

.. code-block:: bash

    $ php app/console propel:sql:build
    $ php app/console propel:sql:insert --force

Теперь в вашей базе данных имеется полнофункциональная таблица ``product`` с колонками,
соответствующими указанной вами схеме. 

.. tip::

    Можно также скомбинировать последние три команды, используя следующую команду:
    ``php app/console propel:build --insert-sql``.

Сохранение объектов в базу данных 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Теперь, когда у вас есть объект ``Product`` и соответствующая ему таблица ``product``,
вы готовы к тому, чтобы сохранять данные в базу данных. Изнутри контроллера это делается
довольно просто. Добавьте следующий метод к ``DefaultController``  в пакете::

    // src/Acme/StoreBundle/Controller/DefaultController.php

    // ...
    use Acme\StoreBundle\Model\Product;
    use Symfony\Component\HttpFoundation\Response;

    public function createAction()
    {
        $product = new Product();
        $product->setName('A Foo Bar');
        $product->setPrice(19.99);
        $product->setDescription('Lorem ipsum dolor');

        $product->save();

        return new Response('Created product id '.$product->getId());
    }

В этом отрывке кода вы создаете экземпляр и работаете с объектом ``$product``.
Когда вы вызываете метод ``save()``, вы этот объект сохраняете в базу данных. 
Никаких других сервисов использовать не нужно, объект сам знает, как ему сохраняться.

.. note::

    If you're following along with this example, you'll need to create a
    :doc:`route <routing>` that points to this action to see it in action.

Fetching Objects from the Database
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Fetching an object back from the database is even easier. For example, suppose
you've configured a route to display a specific ``Product`` based on its ``id``
value::

    // ...
    use Acme\StoreBundle\Model\ProductQuery;

    public function showAction($id)
    {
        $product = ProductQuery::create()
            ->findPk($id);

        if (!$product) {
            throw $this->createNotFoundException(
                'No product found for id '.$id
            );
        }

        // ... do something, like pass the $product object into a template
    }

Updating an Object
~~~~~~~~~~~~~~~~~~

Once you've fetched an object from Propel, updating it is easy. Suppose you
have a route that maps a product id to an update action in a controller::

    // ...
    use Acme\StoreBundle\Model\ProductQuery;

    public function updateAction($id)
    {
        $product = ProductQuery::create()
            ->findPk($id);

        if (!$product) {
            throw $this->createNotFoundException(
                'No product found for id '.$id
            );
        }

        $product->setName('New product name!');
        $product->save();

        return $this->redirect($this->generateUrl('homepage'));
    }

Updating an object involves just three steps:

#. fetching the object from Propel (line 6 - 13);
#. modifying the object (line 15);
#. saving it (line 16).

Deleting an Object
~~~~~~~~~~~~~~~~~~

Deleting an object is very similar to updating, but requires a call to the
``delete()`` method on the object::

    $product->delete();

Querying for Objects
--------------------

Propel provides generated ``Query`` classes to run both basic and complex queries
without any work::

    \Acme\StoreBundle\Model\ProductQuery::create()->findPk($id);

    \Acme\StoreBundle\Model\ProductQuery::create()
        ->filterByName('Foo')
        ->findOne();

Imagine that you want to query for products which cost more than 19.99, ordered
from cheapest to most expensive. From inside a controller, do the following::

    $products = \Acme\StoreBundle\Model\ProductQuery::create()
        ->filterByPrice(array('min' => 19.99))
        ->orderByPrice()
        ->find();

In one line, you get your products in a powerful oriented object way. No need
to waste your time with SQL or whatever, Symfony2 offers fully object oriented
programming and Propel respects the same philosophy by providing an awesome
abstraction layer.

If you want to reuse some queries, you can add your own methods to the
``ProductQuery`` class::

    // src/Acme/StoreBundle/Model/ProductQuery.php
    class ProductQuery extends BaseProductQuery
    {
        public function filterByExpensivePrice()
        {
            return $this
                ->filterByPrice(array('min' => 1000));
        }
    }

But note that Propel generates a lot of methods for you and a simple
``findAllOrderedByName()`` can be written without any effort::

    \Acme\StoreBundle\Model\ProductQuery::create()
        ->orderByName()
        ->find();

Relationships/Associations
--------------------------

Suppose that the products in your application all belong to exactly one
"category". In this case, you'll need a ``Category`` object and a way to relate
a ``Product`` object to a ``Category`` object.

Start by adding the ``category`` definition in your ``schema.xml``:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8" ?>
    <database name="default"
        namespace="Acme\StoreBundle\Model"
        defaultIdMethod="native">
        <table name="product">
            <column name="id"
                type="integer"
                required="true"
                primaryKey="true"
                autoIncrement="true" />

            <column name="name"
                type="varchar"
                primaryString="true"
                size="100" />

            <column name="price"
                type="decimal" />

            <column name="description"
                type="longvarchar" />

            <column name="category_id"
                type="integer" />

            <foreign-key foreignTable="category">
                <reference local="category_id" foreign="id" />
            </foreign-key>
        </table>

        <table name="category">
            <column name="id"
                type="integer"
                required="true"
                primaryKey="true"
                autoIncrement="true" />

            <column name="name"
                type="varchar"
                primaryString="true"
                size="100" />
       </table>
    </database>

Create the classes:

.. code-block:: bash

    $ php app/console propel:model:build

Assuming you have products in your database, you don't want to lose them. Thanks to
migrations, Propel will be able to update your database without losing existing
data.

.. code-block:: bash

    $ php app/console propel:migration:generate-diff
    $ php app/console propel:migration:migrate

Your database has been updated, you can continue writing your application.

Saving Related Objects
~~~~~~~~~~~~~~~~~~~~~~

Now, try the code in action. Imagine you're inside a controller::

    // ...
    use Acme\StoreBundle\Model\Category;
    use Acme\StoreBundle\Model\Product;
    use Symfony\Component\HttpFoundation\Response;

    class DefaultController extends Controller
    {
        public function createProductAction()
        {
            $category = new Category();
            $category->setName('Main Products');

            $product = new Product();
            $product->setName('Foo');
            $product->setPrice(19.99);
            // relate this product to the category
            $product->setCategory($category);

            // save the whole
            $product->save();

            return new Response(
                'Created product id: '.$product->getId().' and category id: '.$category->getId()
            );
        }
    }

Now, a single row is added to both the ``category`` and ``product`` tables. The
``product.category_id`` column for the new product is set to whatever the id is
of the new category. Propel manages the persistence of this relationship for
you.

Fetching Related Objects
~~~~~~~~~~~~~~~~~~~~~~~~

When you need to fetch associated objects, your workflow looks just like it did
before.  First, fetch a ``$product`` object and then access its related
``Category``::

    // ...
    use Acme\StoreBundle\Model\ProductQuery;

    public function showAction($id)
    {
        $product = ProductQuery::create()
            ->joinWithCategory()
            ->findPk($id);

        $categoryName = $product->getCategory()->getName();

        // ...
    }

Note, in the above example, only one query was made.

More information on Associations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You will find more information on relations by reading the dedicated chapter on
`Relationships`_.

Lifecycle Callbacks
-------------------

Sometimes, you need to perform an action right before or after an object is
inserted, updated, or deleted.  These types of actions are known as "lifecycle"
callbacks or "hooks", as they're callback methods that you need to execute
during different stages of the lifecycle of an object (e.g. the object is
inserted, updated, deleted, etc).

To add a hook, just add a new method to the object class::

    // src/Acme/StoreBundle/Model/Product.php

    // ...
    class Product extends BaseProduct
    {
        public function preInsert(\PropelPDO $con = null)
        {
            // do something before the object is inserted
        }
    }

Propel provides the following hooks:

* ``preInsert()`` code executed before insertion of a new object
* ``postInsert()`` code executed after insertion of a new object
* ``preUpdate()`` code executed before update of an existing object
* ``postUpdate()`` code executed after update of an existing object
* ``preSave()`` code executed before saving an object (new or existing)
* ``postSave()`` code executed after saving an object (new or existing)
* ``preDelete()`` code executed before deleting an object
* ``postDelete()`` code executed after deleting an object

Behaviors
---------

All bundled behaviors in Propel are working with Symfony2. To get more
information about how to use Propel behaviors, look at the `Behaviors reference
section`_.

Commands
--------

You should read the dedicated section for `Propel commands in Symfony2`_.

.. _`Working With Symfony2`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2.html#installation
.. _`PropelBundle configuration section`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2.html#configuration
.. _`Relationships`: http://propelorm.org/Propel/documentation/04-relationships.html
.. _`Behaviors reference section`: http://propelorm.org/Propel/documentation/#behaviors-reference
.. _`Propel commands in Symfony2`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2#the-commands
