.. index::
   single: Form; Events

How to Dynamically Modify Forms Using Form Events
===================================================

Before jumping right into dynamic form generation, let's have a quick review
of what a bare form class looks like::

    // src/Acme/DemoBundle/Form/Type/ProductType.php
    namespace Acme\DemoBundle\Form\Type;

    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;

    class ProductType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options)
        {
            $builder->add('name');
            $builder->add('price');
        }

        public function getName()
        {
            return 'product';
        }
    }

.. note::

    If this particular section of code isn't already familiar to you, you
    probably need to take a step back and first review the :doc:`Forms chapter </book/forms>`
    before proceeding.

Let's assume for a moment that this form utilizes an imaginary "Product" class
that has only two relevant properties ("name" and "price"). The form generated
from this class will look the exact same regardless if a new Product is being created
or if an existing product is being edited (e.g. a product fetched from the database).

Suppose now, that you don't want the user to be able to change the ``name`` value
once the object has been created. To do this, you can rely on Symfony's :doc:`Event Dispatcher </components/event_dispatcher/introduction>`
system to analyze the data on the object and modify the form based on the
Product object's data. In this entry, you'll learn how to add this level of
flexibility to your forms.

.. _`cookbook-forms-event-subscriber`:

Adding An Event Subscriber To A Form Class
------------------------------------------

So, instead of directly adding that "name" widget via your ProductType form
class, let's delegate the responsibility of creating that particular field
to an Event Subscriber::

    // src/Acme/DemoBundle/Form/Type/ProductType.php
    namespace Acme\DemoBundle\Form\Type;

    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Acme\DemoBundle\Form\EventListener\AddNameFieldSubscriber;

    class ProductType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options)
        {
            $builder->add('price');

            $builder->addEventSubscriber(new AddNameFieldSubscriber());
        }

        public function getName()
        {
            return 'product';
        }
    }

.. _`cookbook-forms-inside-subscriber-class`:

Inside the Event Subscriber Class
---------------------------------

The goal is to create a "name" field *only* if the underlying Product object
is new (e.g. hasn't been persisted to the database). Based on that, the subscriber
might look like the following:

.. versionadded:: 2.2
    The ability to pass a string into :method:`FormInterface::add<Symfony\\Component\\Form\\FormInterface::add>`
    was added in Symfony 2.2.

.. code-block:: php

    // src/Acme/DemoBundle/Form/EventListener/AddNameFieldSubscriber.php
    namespace Acme\DemoBundle\Form\EventListener;

    use Symfony\Component\Form\FormEvent;
    use Symfony\Component\Form\FormEvents;
    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class AddNameFieldSubscriber implements EventSubscriberInterface
    {
        public static function getSubscribedEvents()
        {
            // Tells the dispatcher that you want to listen on the form.pre_set_data
            // event and that the preSetData method should be called.
            return array(FormEvents::PRE_SET_DATA => 'preSetData');
        }

        public function preSetData(FormEvent $event)
        {
            $data = $event->getData();
            $form = $event->getForm();

            // During form creation setData() is called with null as an argument
            // by the FormBuilder constructor. You're only concerned with when
            // setData is called with an actual Entity object in it (whether new
            // or fetched with Doctrine). This if statement lets you skip right
            // over the null condition.
            if (null === $data) {
                return;
            }

            // check if the product object is "new"
            if (!$data->getId()) {
                $form->add('name', 'text');
            }
        }
    }

.. caution::

    It is easy to misunderstand the purpose of the ``if (null === $data)`` segment
    of this event subscriber. To fully understand its role, you might consider
    also taking a look at the `Form class`_ and paying special attention to
    where setData() is called at the end of the constructor, as well as the
    setData() method itself.

The ``FormEvents::PRE_SET_DATA`` line actually resolves to the string ``form.pre_set_data``.
The `FormEvents class`_ serves an organizational purpose. It is a centralized location
in which you can find all of the various form events available.

While this example could have used the ``form.post_set_data``
event just as effectively, by using ``form.pre_set_data`` you guarantee that
the data being retrieved from the ``Event`` object has in no way been modified
by any other subscribers or listeners because ``form.pre_set_data`` is the
first form event dispatched.

.. note::

    You may view the full list of form events via the `FormEvents class`_,
    found in the form bundle.

.. _`FormEvents class`: https://github.com/symfony/Form/blob/master/FormEvents.php
.. _`Form class`: https://github.com/symfony/symfony/blob/master/src/Symfony/Component/Form/Form.php