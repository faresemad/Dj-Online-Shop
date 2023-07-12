# Dj-Online-Shop
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Account: LinkedIn](https://img.shields.io/badge/Fares%20Emad-LinkedIn-blue)](https://www.linkedin.com/in/faresemad/)
[![Account: Facebook](https://img.shields.io/badge/Fares%20Emad-Facebook-blue)](https://www.facebook.com/faresemadx)

A functioning online e-shop that uses the Django framework to build the web application.

## Getting Started
- Clone the repository
```bash
$ git clone https://github.com/faresemad/Dj-Online-Shop.git
```
- Install the requirements
```bash
$ pip install -r requirements.txt
```
- Run the server
```bash
$ python manage.py runserver
```
- Open the browser and go to `http://localhost:8000/`


# Using Django sessions
- To use sessions, you have to make sure that the **MIDDLEWARE** setting of your project contains `django.contrib.sessions.middleware.SessionMiddleware`.
- The session dictionary accepts any Python object by default that can be serialized to JSON.
- You can set a variable in the session like this:
```python
request.session['foo'] = 'bar'
```
- Retrieve a session key as follows:
```python
request.session.get('foo')
```
- Delete a key you previously stored in the session as follows:
```python
del request.session['foo']
```

> When users log in to the site, their anonymous session is lost, and a new session is created for authenticated users. If you store items in an anonymous session that you need to keep after the user logs in, you will have to copy the old session data into the new session. You can do this by retrieving the session data before you log in the user using the login() function of the Django authentication system and storing it in the session after that.

## Session settings
> There are several settings you can use to configure sessions for your
> project. The most important is **SESSION_ENGINE**. This setting allows you
> to set the place where sessions are stored. By default, Django stores
> sessions in the database using the Session model of the
> `django.contrib.sessions` application.

- Edit the settings.py file of your project and add the following setting to it:
```python
CART_SESSION_ID = 'cart'
```
- This is the key that you are going to use to store the cart in the user session. Since Django sessions are managed per visitor, you can use the same cart session key for all sessions.

### Creating a shopping cart class
- Create a file called `cart.py` in the `shop` application directory.
- In this file, you are going to create a `Cart` class that will be responsible for managing the shopping cart.
- The `Cart` class will be initialized with the current request object, which will be used to access the current session.
- The `Cart` class will have the following attributes and methods:
    - `cart = request.session.get(settings.CART_SESSION_ID)` - get the current cart from the session.
    - `request.session[settings.CART_SESSION_ID] = self.cart` - save the current cart in the session.
    - `add(product, quantity=1, override_quantity=False)` - add a product to the cart or update its quantity.
    - `remove(product)` - remove a product from the cart.
    - `__iter__()` - iterate over the items in the cart and get the products from the database.
    - `__len__()` - return the total number of items stored in the cart.
    - `get_total_price()` - return the total cost of the items in the cart.
    - `clear()` - remove the cart from the session.
- The `Cart` class will be used as follows:
```python
from decimal import Decimal
from django.conf import settings
from shop.models import Product


class Cart:
    def __init__(self, request):
        self.session = request.session
        cart = self.session.get(settings.CART_SESSION_ID)
        if not cart:
            # save an empty cart in the session
            cart = self.session[settings.CART_SESSION_ID] = {}
        self.cart = cart

    def add(self, product, quantity=1, override_quantity=False):
        """
        Add a product to the cart or update its quantity.
        """
        product_id = str(product.id)
        if product_id not in self.cart:
            self.cart[product_id] = {"quantity": 0, "price": str(product.price)}
        
        if override_quantity:
            self.cart[product_id]["quantity"] = quantity
        else:
            self.cart[product_id]["quantity"] += quantity
        self.save()
    
    def save(self):
        # mark the session as "modified" to make sure it gets saved
        self.session.modified = True
    
    def remove(self, product):
        product_id = str(product.id)
        if product_id in self.cart:
            del self.cart[product_id]
            self.save()
    
    def __iter__(self):
        product_ids = self.cart.keys()
        # get the product objects and add them to the cart
        products = Product.objects.filter(id__in=product_ids)
        cart = self.cart.copy()
        for product in products:
            cart[str(product.id)]["product"] = product
        for item in cart.values():
            item["price"] = Decimal(item["price"])
            item["total_price"] = item["price"] * item["quantity"]
            yield item
    
    def __len__(self):
        """
        Count all items in the cart.
        """
        return sum(item["quantity"] for item in self.cart.values())
    
    def get_total_price(self):
        return sum(Decimal(item["price"]) * item["quantity"] for item in self.cart.values())

    def clear(self):
        # remove cart from session
        del self.session[settings.CART_SESSION_ID]
        self.save()
```
#### The `__init__()` method
- The `__init__()` method initializes the cart.
#### The `add()` method
- The `add()` method adds a product to the cart or updates its quantity.
#### The `save()` method
- The `save()` method marks the session as modified to make sure it gets saved.
#### The `remove()` method
- The `remove()` method removes a product from the cart.
#### The `__iter__()` method
- The `__iter__()` method iterates over the items in the cart and gets the products from the database.
#### The `__len__()` method
- The `__len__()` method returns the total number of items stored in the cart.
#### The `get_total_price()` method
- The `get_total_price()` method returns the total cost of the items in the cart.
#### The `clear()` method
- The `clear()` method removes the cart from the session.
#### The `save()` method
- The `save()` method marks the session as modified to make sure it gets saved.


## Creating a context processor for the current cart
- Create a new file inside the cart application directory and name it `context_processors.py`. Context processors can reside anywhere in your code but creating them here will keep your code well organized. Add the following code to the file:
```python
from .cart import Cart
def cart(request):
    return {'cart': Cart(request)}
```
- Edit the settings.py file of your project and add `cart.context_processors.cart` to the `context_processors` option inside the **TEMPLATES** setting.

## Using Django with Celery and RabbitMQ

### Installing Celery
- Let’s install Celery and integrate it into the project. Install Celery via pip using the following command:
```bash
$ pip install celery==5.2.7
```
### Installing RabbitMQ
- Celery requires a messaging agent to run. We will use RabbitMQ as the messaging agent. To install RabbitMQ, follow the instructions in the [RabbitMQ installation guide](https://www.rabbitmq.com/download.html).
- After installing Docker on your machine, you can easily pull the RabbitMQ Docker image by running the following command from the shell:
```bash
$ docker pull rabbitmq
```
- If you want to install RabbitMQ natively on your machine instead of using Docker, you will find detailed installation guides for different operating systems at [https://www.rabbitmq.com/download.html](https://www.rabbitmq.com/download.html).
- Execute the following command in the shell to start the RabbitMQ server with Docker:
```bash
$ docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```
- With this command, we are telling RabbitMQ to run on port 5672, and we are running its web-based management user interface on port 15672.
- You will see output that includes the following lines:
    ```bash
    Starting broker...
    ...
    completed with 4 plugins.
    Server startup complete; 4 plugins started.
    ```
- RabbitMQ is running on port 5672 and ready to receive messages.
### Accessing RabbitMQ’s management interface
- Open http://127.0.0.1:15672/ in your browser. You will see the login screen for the management UI of RabbitMQ.
- Enter `guest` as both the username and the password and click on **Login**.
- If you use RabbitMQ in a production environment, you will need to create a new admin user and remove the default guest user. You can do that in the Admin section of the management UI.
### Adding Celery to your project
- Create a new file next to the `settings.py` file of myshop and name it `celery.py`. This file will contain the **Celery** configuration for your project. Add the following code to it:
```python
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myshop.settings")
app = Celery("myshop")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```
- You need to import the celery module in the `__init__.py` file of your project to ensure it is loaded when Django starts.
- Edit the **myshop/__init__.py** file and add the following code to it:
```python
# import celery
from .celery import app as celery_app

__all__ = ["celery_app"]
```

### Running a Celery worker
- A Celery worker is a process that handles bookkeeping features like sending/receiving queue messages, registering tasks, killing hung tasks, tracking status, etc. A worker instance can consume from any number of message queues.
- Open another shell and start a Celery worker from your project directory, using the following command:
```bash
$ celery -A myshop worker -l info
```
- Open `http://127.0.0.1:15672/` in your browser to access the RabbitMQ management UI. You will now see a graph under **Queued messages** and another graph under **Message rates**.

## Adding asynchronous tasks to your application
- Create a new file inside the orders application and name it `tasks.py`. This is the place where Celery will look for asynchronous tasks. Add the following code to it:
```python
from celery import shared_task
from django.core.mail import send_mail
from .models import Order
from django.conf import settings

@shared_task
def order_created(order_id):
    """
    Task to send an e-mail notification when an order is
    successfully created.
    """
    order = Order.objects.get(id=order_id)
    subject = f"Order nr. {order.id}"
    message = (
        f"Dear {order.first_name},\n\n"
        f"You have successfully placed an order."
        f"Your order ID is {order.id}."
    )
    mail_sent = send_mail(subject, message, settings.EMAIL_HOST_USER, [order.email])
    return mail_sent
```
- Now you have to add the task to your `order_create` view. Edit the `views.py` file of the orders application, import the task, and call the `order_created` asynchronous task after clearing the cart, as follows:
```python
def order_create(request):
    # ...
    if request.method == 'POST':
        # ...
        if form.is_valid():
            # ...
            cart.clear()
            # launch asynchronous task
            order_created.delay(order.id)
        # ...
```
- Now, when an order is successfully created, the `order_created` task will be executed asynchronously by the Celery worker.
- You call the `delay()` method of the task to execute it asynchronously. The task will be added to the message queue and executed by the Celery worker as soon as possible.
- Make sure RabbitMQ is running. Then, stop the Celery worker process and start it again with the following command:
```bash
$ celery -A myshop worker -l info
```
- Open http://127.0.0.1:8000/ in your browser, add some products to your shopping cart, and complete an order. In the shell where you started the Celery worker you will see output similar to the following:
```bash
[2022-02-03 20:25:19,569: INFO/MainProcess] Task orders.tasks.order_
created[a94dc22e-372b-4339-bff7-52bc83161c5c] received
...
[2022-02-03 20:25:19,605: INFO/ForkPoolWorker-8] Task orders.tasks.
order_created[a94dc22e-372b-4339-bff7-52bc83161c5c] succeeded in
0.015824042027816176s: 1
```

## Monitoring Celery with Flower
- Install Flower using the following command:
```bash
$ pip install flower==1.1.0
```
- Once installed, you can launch Flower by running the following command in a new shell from your project directory:
```bash
$ celery -A myshop flower
```
- Open `http://localhost:5555/dashboard` in your browser. You will be able to see the active Celery workers and asynchronous task statistics.
