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

    Если вы хотите проделать этот пример вместе с нами, вам нужно будет создать 
    :doc:`route <routing>`, указывающий на это действие.
    
Извлечение объектов из базы данных
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Извлечь объект из базы данных обратно еще проще. Например, допустим, вы настроили 
мрашрут для отображения определенного объекта ``Product``, основываясь на 
значении ``id``::

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

        // ... сделайте что-нибудь, например, передайте объект $product в какой-нибудь шаблон
    }

Обновление объекта
~~~~~~~~~~~~~~~~~~

Раз уж вы извлекли объект из Propel, то обновить его сможете легко. Допустим, у вас
есть маршрут, связывающий  product id с действием по обновлению в контроллере::

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

Обновление объекта включает в себя три шага:

#. извлечение объекта из (строки 6 - 13);
#. изменение объекта (строки 15);
#. его сохранение (строки 16).

Удаление объекта
~~~~~~~~~~~~~~~~~~

Удаление объекта очень похоже на обновление, но требует вызова метода
``delete()`` для объекта::

    $product->delete();

Запрос объектов
--------------------

Propel предоставляет сгенерированные классы ``Query`` для запуска базовых и сложных
запросов безо всяких усилий::

    \Acme\StoreBundle\Model\ProductQuery::create()->findPk($id);

    \Acme\StoreBundle\Model\ProductQuery::create()
        ->filterByName('Foo')
        ->findOne();

Представьте себе, что вы хотите запросить продукты, которые стоят больше 19.99, 
в порядке возрастания цены. Находясь внутри контроллера, сделайте следующее::

    $products = \Acme\StoreBundle\Model\ProductQuery::create()
        ->filterByPrice(array('min' => 19.99))
        ->orderByPrice()
        ->find();

Короче говоря, вы поулчаете свои продукты мощным объектно-ориентированным способом. Нет нужды
тратить свое время на SQL или ему подобное, так как Symfony2  предоставляет полностью объектно 
ориентированное программирование, а Propel исповедует ту же философию, предоставляя впечатляющий
уровень абстракции.

Если вы хотите использовать некоторые запросы повторно, можете добавить  ваши собственные методы
к классу ``ProductQuery``::

    // src/Acme/StoreBundle/Model/ProductQuery.php
    class ProductQuery extends BaseProductQuery
    {
        public function filterByExpensivePrice()
        {
            return $this
                ->filterByPrice(array('min' => 1000));
        }
    }

Но учтите, что Propel генерирует множество методов для вас и можно написать, к примеру, 
простой ``findAllOrderedByName()``, причем без всяких усилий::

    \Acme\StoreBundle\Model\ProductQuery::create()
        ->orderByName()
        ->find();

Отношения/Ассоциации (Relationships/Associations)
--------------------------

Допустим, что продукты в вашем приложении все принадлежат к одной категории 
("category"). В этом случае, вам нужен объект ``Category`` и способ соотнести объект 
 ``Product`` с объектом ``Category``.

Начнем с того, что добавим определение ``category`` в ваш файл ``schema.xml``:

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

Создайте классы:

.. code-block:: bash

    $ php app/console propel:model:build

Предположительно, у вас есть продукты в базе данных, и вы не хотите их потерять. Благодаря 
миграциям (migrations), Propel сможет обновить вашу базу данных, не теряя существующие данные.

.. code-block:: bash

    $ php app/console propel:migration:generate-diff
    $ php app/console propel:migration:migrate

Ваша база данных была обновлена, и вы можете продолжать писать свое приложение.

Сохранение взаимосвязанных объектов (Saving Related Objects)
~~~~~~~~~~~~~~~~~~~~~~

Теперь, давайте попробуем код в действии. Представьте себе, что вы внутри контроллера::

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

Итак, единтсвенная строка была добавлена в таблицы ``category`` и ``product``. Колонка
``product.category_id`` для нового продукта получила значение для id новой категории. 
В итоге Propel управляет сохранением этого отношения вместо вас. 

Извлечение взаимосвязанных объектов 
~~~~~~~~~~~~~~~~~~~~~~~~

Когда вам нужно извлечь из базы данных взаимосвязанные объекты, ваш рабочий процесс выглядит 
так же, как и раньше. Для начала, излечем объект ``$product`` и потом получим доступ 
к вазимосвязанным объектом  ``Category``::

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

Заметьте, что в приведенном выше примере был послан только один запрос.

Больше информации об ассоциациях
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Больше информации об отношениях можно найти в специально посвященной им главе  `Relationships`_.

Обратный вызов жизненного цикла (Lifecycle Callbacks)
-------------------

Иногда, вам нужно выполнить действие непосредственно до или после того, как 
объект вставлен, обновлен, или удален. Такой тип действий называется обратным вызовом
"жизненного цикла" или  привязками "hooks", так как они являются методами обратного вызова 
(callback methods), которые бывает необходимо выполнить на различных стадиях жизненноо цикла объекта
(т.е. когда объект втавляется, обновляется, удаляется, и т.д.).

Чтобы добавить привязку, просто добавьте новый метод к классу объекта::

    // src/Acme/StoreBundle/Model/Product.php

    // ...
    class Product extends BaseProduct
    {
        public function preInsert(\PropelPDO $con = null)
        {
            // сделайте что-нибудь до того, как объект будет вставлен
        }
    }

Propel предоставляет следующие привязки:

* ``preInsert()`` код выполняется до вставки нового объекта 
* ``postInsert()`` код выполняется после вставки нового объекта
* ``preUpdate()`` код выполняется до обновления нового объекта 
* ``postUpdate()`` код выполняется после обновления вставки нового объекта
* ``preSave()`` код выполняется до сохранения объекта (нового или существующего)
* ``postSave()`` код выполняется после сохранения объекта (нового или существующего)
* ``preDelete()`` код выполняется до удаления объекта
* ``postDelete()`` код выполняется после сохранения объекта

Формы поведения (Behaviors)
---------

Все прилагаемые с Propel формы поведения работают с Symfony2. Больше информации о том, как
использовать формы поведения Propel можно найти в `Behaviors reference section`_.

Команды
--------

Вам следует прочесть посвященный им раздел в `Propel commands in Symfony2`_.

.. _`Working With Symfony2`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2.html#installation
.. _`PropelBundle configuration section`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2.html#configuration
.. _`Relationships`: http://propelorm.org/Propel/documentation/04-relationships.html
.. _`Behaviors reference section`: http://propelorm.org/Propel/documentation/#behaviors-reference
.. _`Propel commands in Symfony2`: http://propelorm.org/Propel/cookbook/symfony2/working-with-symfony2#the-commands
