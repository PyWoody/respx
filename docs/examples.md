# Test Case Examples

Here's some test case examples, not exactly *how-to*, but to be inspired from.

## pytest

### Built-in Fixture

RESPX includes the `respx_mock` pytest httpx *fixture*.

``` python
import httpx


def test_fixture(respx_mock):
    respx_mock.get("https://foo.bar/").mock(return_value=httpx.Response(204))
    response = httpx.get("https://foo.bar/")
    assert response.status_code == 204
```

### Built-in Marker

To configure the `respx_mock` fixture, use the `respx` *marker*.

``` python
import httpx
import pytest


@pytest.mark.respx(base_url="https://foo.bar")
def test_configured_fixture(respx_mock):
    respx_mock.get("/baz/").mock(return_value=httpx.Response(204))
    response = httpx.get("https://foo.bar/baz/")
    assert response.status_code == 204
```

> See router [configuration](api.md#configuration) reference for more details.


### Custom Fixtures
``` python
# conftest.py
import pytest
import respx

from httpx import Response


@pytest.fixture
def mocked_api():
    with respx.mock(
        base_url="https://foo.bar", assert_all_called=False
    ) as respx_mock:
        users_route = respx_mock.get("/users/", name="list_users")
        users_route.return_value = Response(200, json=[])
        ...
        yield respx_mock
```

``` python
# test_api.py
import httpx


def test_list_users(mocked_api):
    response = httpx.get("https://foo.bar/users/")
    assert response.status_code == 200
    assert response.json() == []
    assert mocked_api["list_users"].called
```

**Session Scoped Fixtures**

If a session scoped RESPX fixture is used in an async context, you also need to broaden the `pytest-asyncio`
 [event_loop](https://github.com/pytest-dev/pytest-asyncio#event_loop) fixture.
 You can use the `session_event_loop` utility for this. 

``` python
# conftest.py
import pytest
import respx

from respx.fixtures import session_event_loop as event_loop  # noqa: F401


@pytest.fixture(scope="session")
async def mocked_api(event_loop):  # noqa: F811
    async with respx.mock(base_url="https://foo.bar") as respx_mock:
        ...
        yield respx_mock
```

### Async Test Cases
``` python
import httpx
import respx


@respx.mock
async def test_async_decorator():
    async with httpx.AsyncClient() as client:
        route = respx.get("https://example.org/")
        response = await client.get("https://example.org/")
        assert route.called
        assert response.status_code == 200


async def test_async_ctx_manager():
    async with respx.mock:
        async with httpx.AsyncClient() as client:
            route = respx.get("https://example.org/")
            response = await client.get("https://example.org/")
            assert route.called
            assert response.status_code == 200
```


### Multiple Side Effects

RESPX and httpx can make it easy to test conditions that require multiple side effects.

```python
class PizzaFactory:

    def __init__(self, cheese=0, pepperonis=0, jalapenos=0, dough=0, sauce=0):
        self.cheese = int(cheese)
        self.jalapenos = int(jalapenos)
        self.dough = int(dough)
        self.sauce = int(sauce)
        self.pepperonis = int(pepperonis)
        self.pizzas_sold = 0

    def __call__(self, request, route):
        status = {
            'pizzas_sold': self.pizzas_sold,
            'remaining_ingredients': {
                'jalapenos': self.jalapenos,
                'cheese': self.cheese,
                'dough': self.dough,
                'pepperonis': self.pepperonis,
                'sauce': self.sauce,
            }
        }
        return httpx.Response(200, json=status)

    def cheese_pizza(self, request, route, amount=1, jalapenos=False):
        amount = int(request.url.params.get('amount', amount))
        jalapenos = bool(request.url.params.get('jalapenos', jalapenos))
        ingredients = {self.cheese, self.sauce, self.dough}
        if jalapenos:
            ingredients.add(self.jalapenos)
        if any(i - amount < 0 for i in ingredients):
            return httpx.Response(400)
        self.cheese -= amount
        self.sauce -= amount
        self.dough -= amount
        self.pizzas_sold += amount
        if jalapenos:
            self.jalapenos -= amount
        return httpx.Response(201)

    def pepperoni_pizza(self, request, route, amount=1, jalapenos=False):
        amount = int(request.url.params.get('amount', amount))
        jalapenos = bool(request.url.params.get('jalapenos', jalapenos))
        ingredients = {self.cheese, self.sauce, self.dough, self.pepperonis}
        if jalapenos:
            ingredients.add(self.jalapenos)
        if any(i - amount < 0 for i in ingredients):
            return httpx.Response(400)
        self.cheese -= amount
        self.pepperonis -= amount
        self.sauce -= amount
        self.dough -= amount
        if jalapenos:
            self.jalapenos -= amount
        self.pizzas_sold += amount
        return httpx.Response(201)


def test_pizza_side_effects(respx_mock):
    pizza_factory = PizzaFactory(
        cheese=30, dough=30, sauce=30, pepperonis=10, jalapenos=5
    )
    status_route = respx_mock.get('https://example.org/status')
    status_route.side_effect = pizza_factory

    cheese_route = respx_mock.post('https://example.org/order/cheese')
    cheese_route.side_effect = pizza_factory.cheese_pizza

    pepperoni_route = respx_mock.post('https://example.org/order/pepperoni')
    pepperoni_route.side_effect = pizza_factory.pepperoni_pizza

    # Assert orders for more than the current ingredients fail
    # and do not alter state
    bad_order = httpx.post(
        'https://example.org/order/pepperoni',
        params={'jalapenos': True, 'amount': 100}
    )
    assert bad_order.status_code == 400
    assert pizza_factory.pizzas_sold == 0
    assert pizza_factory.dough == 30

    # Assert one insufficent ingredient causes order failure
    bad_order = httpx.post(
        'https://example.org/order/cheese',
        params={'jalapenos': True, 'amount': 6}
    )
    assert bad_order.status_code == 400
    assert pizza_factory.pizzas_sold == 0
    assert pizza_factory.jalapenos == 5

    # Order a cheese pizza
    order = httpx.post(
        'https://example.org/order/cheese', params={'jalapenos': True}
    )
    assert order.status_code == 201
    assert pizza_factory.pizzas_sold == 1
    assert pizza_factory.sauce == 29
    assert pizza_factory.dough == 29
    assert pizza_factory.cheese == 29
    assert pizza_factory.jalapenos == 4

    # Order 10 pepperoni pizzas
    order = httpx.post(
        'https://example.org/order/pepperoni', params={'amount': 10}
    )
    assert order.status_code == 201
    assert pizza_factory.pizzas_sold == 11
    assert pizza_factory.sauce == 19
    assert pizza_factory.dough == 19
    assert pizza_factory.cheese == 19
    assert pizza_factory.pepperonis == 0

    # Assert status
    status = httpx.get('https://example.org/status')
    assert status.json()['remaining_ingredients']['sauce'] == pizza_factory.sauce
    assert status.json()['remaining_ingredients']['dough'] == pizza_factory.dough
    assert status.json()['remaining_ingredients']['cheese'] == pizza_factory.cheese
    assert status.json()['remaining_ingredients']['pepperonis'] == pizza_factory.pepperonis
    assert status.json()['remaining_ingredients']['jalapenos'] == pizza_factory.jalapenos
    assert status.json()['pizzas_sold'] == pizza_factory.pizzas_sold
```

## unittest

### Regular Decoration

``` python
# test_api.py
import httpx
import respx
import unittest


class APITestCase(unittest.TestCase):
    @respx.mock
    def test_some_endpoint(self):
        respx.get("https://example.org/") % 202
        response = httpx.get("https://example.org/")
        self.assertEqual(response.status_code, 202)
```

### Reuse SetUp & TearDown

``` python
# testcases.py
import respx

from httpx import Response


class MockedAPIMixin:
    @classmethod
    def setUpClass(cls):
        cls.mocked_api = respx.mock(
            base_url="https://foo.bar", assert_all_called=False
        )
        users_route = cls.mocked_api.get("/users/", name="list_users")
        users_route.return_value = Response(200, json=[])
        ...

    def setUp(self):
        self.mocked_api.start()
        self.addCleanup(self.mocked_api.stop)
```
``` python
# test_api.py
import httpx
import unittest

from .testcases import MockedAPIMixin


class APITestCase(MockedAPIMixin, unittest.TestCase):
    def test_list_users(self):
        response = httpx.get("https://foo.bar/users/")
        self.assertEqual(response.status_code, 200)
        self.assertListEqual(response.json(), [])
        self.assertTrue(self.mocked_api["list_users"].called)
```

### Async Test Cases
``` python
import asynctest
import httpx
import respx


class MyTestCase(asynctest.TestCase):
    @respx.mock
    async def test_async_decorator(self):
        async with httpx.AsyncClient() as client:
            route = respx.get("https://example.org/")
            response = await client.get("https://example.org/")
            assert route.called
            assert response.status_code == 200

    async def test_async_ctx_manager(self):
        async with respx.mock:
            async with httpx.AsyncClient() as client:
                route = respx.get("https://example.org/")
                response = await client.get("https://example.org/")
                assert route.called
                assert response.status_code == 200
```
