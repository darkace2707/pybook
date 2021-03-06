# Лабораторная работа №13. MicroSlack

Эта лабораторная работа посвящена созданию простого клона мессенджера Slack.

```python
from aiohttp import web

app = web.Application()
web.run_app(app, host='127.0.0.1', port=8080)
```

```python
from aiohttp import web

async def index(request):
    return web.Response(text='Index page')

app = web.Application()
app.router.add_route('GET', '/', index)
web.run_app(app, host='127.0.0.1', port=8080)
```


```python
from aiohttp import web
import jinja2
import aiohttp_jinja2

@aiohttp_jinja2.template('index.html')
async def index(request):
    return {'title': 'Index page'}

app = web.Application()
aiohttp_jinja2.setup(app, loader=jinja2.FileSystemLoader('templates'))
app.router.add_get('/', index)
web.run_app(app, host='127.0.0.1', port=8080)
```

```html
<!DOCTYPE html>                                                             
<html>
    <head>
        <title>{{ title }}</title>
    </head>
    <body>
        <h1>Welcome to {{ title }}</h1>
    </body>
</html>
```

```python
from aiohttp import web
import jinja2
import aiohttp_jinja2

from chat.views import Index

async def create_app():
    app = web.Application()
    aiohttp_jinja2.setup(app, loader=jinja2.FileSystemLoader('templates'))
    app.router.add_static('/static', 'static', name='static') # аргумент name используется для того, чтобы мы могли обратиться к app.router.static
    app.router.add_get('/', Index)
    return app

loop = asyncio.get_event_loop()
app = loop.run_until_complete(create_app())
web.run_app(app, host='127.0.0.1', port=8080)
```

```python
from aiohttp import web
import aiohttp_jinja2


class Index(web.View):
    
    @aiohttp_jinja2.template('index.html')
    async def get(self):
        return {'title': 'Index page'}
```


```html
...
<head>
    <title>{{ title }}</title>
    <script src="{{ app.router.static.url(filename='chat.js') }}"></script>
</head>
...
```


```python
from aiohttp import web
import aiohttp_jinja2


class LoginView(web.View):
    # http://aiohttp.readthedocs.io/en/stable/web.html#class-based-views
    
    @aiohttp_jinja2.template('login.html')
    async def get(self):
        if self.request.cookies.get('user'):
            return web.HTTPFound('/')
        return {'title': 'Login'}
    
    async def post(self):
        response = web.HTTPFound('/')
        # http://aiohttp.readthedocs.io/en/stable/web.html#http-forms
        data = await self.request.post()
        response.set_cookie('user', data['name'])
        return response
```


```html
<!DOCTYPE html>
<html>
    <head>
        <title>{{ title }}</title>
        <link rel="stylesheet" href="//oss.maxcdn.com/semantic-ui/2.1.8/semantic.min.css" type="text/css">
    </head>
    <body>
        <div class="ui three column stackable centered grid">                       
            <div class="column">
                <form action="/login" method="post">
                    <div class="ui fluid action input">
                        <input type="text" name="name" placeholder="Username" required>
                        <button type="submit" class="ui primary button">Log in</button>
                    </div>
                </form>
            </div>
        </div>
    </body>
</html>
```

```python
...
from accounts.views import Login
...

async def create_app():
    ...
    app.router.add_route('*', '/login', Login)
    ...
```


```python
from aiohttp import web

async def auth_cookie_factory(app, handler):
    # http://aiohttp.readthedocs.io/en/stable/web.html#middlewares
    async def middleware(request):
        # http://aiohttp.readthedocs.io/en/stable/web_reference.html#aiohttp.web.BaseRequest.path
        # http://aiohttp.readthedocs.io/en/stable/web_reference.html#aiohttp.web.BaseRequest.cookies
        if request.path != '/login' and request.cookies.get('user') is None:
            # http://aiohttp.readthedocs.io/en/stable/web.html#exceptions
            return web.HTTPFound('/login')
        return await handler(request)
    return middleware
```

```python
...
from middlewares import auth_cookie_factory
...

async def create_app():
    app = web.Application(middlewares=[auth_cookie_factory,])
    ...
```

#### Замена кук на сессии:

```python
from aiohttp import web
import jinja2
import aiohttp_jinja2
import aioredis
from aiohttp_session import session_middleware
from aiohttp_session.redis_storage import RedisStorage

from chat.views import Index
from accounts.views import Login
from middlewares import auth_session_factory, request_session_factory

async def create_app():
    async def close_redis(app):
        app.redis_pool.close()
        await app.redis_pool.wait_closed()
    
    redis_pool = await aioredis.create_pool(('127.0.0.1', 6379), loop=loop)
    app = web.Application(middlewares=[
        session_middleware(RedisStorage(redis_pool)),
        request_session_factory,
        auth_session_factory
    ])
    app.redis_pool = redis_pool
    aiohttp_jinja2.setup(app, loader=jinja2.FileSystemLoader('templates'))
    app.router.add_static('/static', 'static', name='static')
    app.router.add_get('/', Index)
    app.router.add_route('*', '/login', Login)
    app.on_shutdown.append(close_redis)
    return app

loop = asyncio.get_event_loop()
app = loop.run_until_complete(create_app())
web.run_app(app, host='127.0.0.1', port=8080)
```


```python
from aiohttp import web
from aiohttp_session import get_session

async def request_session_factory(app, handler):
    async def middleware(request):
        request.session = await get_session(request)
        return await handler(request)
    return middleware

async def auth_session_factory(app, handler):
    async def middleware(request):
        if request.path != '/login' and request.session.get('user') is None:
            return web.HTTPFound('/login')
        return await handler(request)
    return middleware
```

```python
...
from aiohttp_session import get_session

class Login(web.View):

    @aiohttp_jinja2.template('login.html')
    async def get(self):
        if self.request.session.get('user'):
        ...
    
    async def post(self):
        ...
        self.request.session['user'] = data['name']
        ...
```

#### Шаблон интерфейса

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <title></title>
        <link href="https://fonts.googleapis.com/css?family=Roboto:300,400,700,900&amp;subset=cyrillic,cyrillic-ext" rel="stylesheet">
        <link href="{{ app.router.static.url(filename='css/style.css') }}" rel='stylesheet' type='text/css'>
        <script src="{{ app.router.static.url(filename='js/vue.js') }}"></script>
    <body>
        <div id="app">
            <app-header></app-header>
            <div class="main">
                <channels></channels>
                <messages></messages>
            </div>
            <app-footer></app-footer>
        </div>
    <script src="{{ app.router.static.url(filename='js/chat.js') }}"></script>
    </body>
</html>
```

```js
Vue.component('app-header', {
    template: `
        <div class="header">
            <div class="team-menu">MicroSlack</div>
            <div class="channel-menu">
                <span class="channel-menu_name">
                    <span class="channel-menu_prefix">#</span>
                    general
                </span>
            </div>
        </div>
    `
})


Vue.component('app-footer', {
    template: `
        <div class="footer">
            <div class="user-menu">
                <span class="user-menu_profile-pic"></span>
                <span class="user-menu_username">scotch</span>
                <img class="connection_icon" src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAMAAABEpIrGAAABmFBMVEUAAAD////////////////////////////////////2+/LR5bKw1Hmfy1KUxz2VyD2izVKz1nnS5rP////A3JuOw0qKwkCNxD+QxT6Sxj6Txz6SxUnC3Jv1+fGXx2GDvkCGwECIwUCLwj+PxD6PxT+JwUCFwECZyGD2+vGSxWF9vEGAvkGDv0CMwz+Wx2GPw2F4ukJ7u0J+vUGBvkGHwUB8u0KSxGG31pp0uEN3uUJ5u0KFv0CCv0B6u0K415p5uU1yt0N/vUF1uEN8u0zG3bFttURwtkR5ukLH3rGWxnlqtERutUR2uUOZx3l6uVZos0VvtkRxt0Nzt0N8ulVisUVlskVns0VzuENmskVfsEVps0VztlZer0VhsEVjsUVstER1t1aOwXhcrkZdr0VgsEaQwnm/2a9YrUZbrka/2rDz+PFhr09XrEZksE6pzplUq0ZVrEZarUaqzpl0tWJRq0dWrEZ1tmJztWJOqUdSq0dxtGJMqEdNqUdQqkdytWKmzJhXrFBKqEdZrU+716+GvXhjr1dIp0hkr1dYtVOVAAAAFHRSTlMAV8/v/wCH+x/n////////////9kvBHZAAAAG7SURBVHgBvdOxjtNAEIDhGe/MZO3sxVaiIJkiSNdQUPJOeQlqXoCCIg/EU9BQHRKg5CT7ErzrHTa+aBOqaxC/tdLK+2kbj+H/hoWhlCmQr0HeyYxyM8mvkWHKoAfBS6cBWEeYugAzf4QGp1SV8DvU/ZjBdN7iud6hdnOTdl+TuALyrUPEwfdu3nc1ipr9AwdIFZPysJylRDfa6cZL2rfgMd9QjO8R0Y+/u7sa4LHZz4wN/MXEyw1hbK1VZdV7PZ1OyufzktsxXADCW5EkXq06Paan02Uoo3kHmAEzJ8HBN6v5qlkqaxTmCdAzQK8Noi6rXwCrJyutepUMAARnXS++3cvm2xvftR0PzAyQAXtwdNChifvFHppBdR003IDCIg6JDOse4DX8WIdo1TwfpaUgqWC9c4eqqg5HF20QZdAMmDlasdHWkrKR03J0A4iIXRTrpba29laiY8YMyOyMKYkXroyROZZuwVTyztAFJPmZKBGq+FxFVBr5BHr7ubd3GICfAM+88qDHHYe/BmbbIAaGKU/Fz10emDxyHxBhgJTg+DGP3O3QbltMBkd92F2H9sWxB772wo9z2z8FfwDHWbdKLDfq1AAAAABJRU5ErkJggg==">
                <span class="connection_status">online</span>
            </div>
            <div class="input-box">
                <input type="text" class="input-box_text">
            </div>
        </div>
    `
})


Vue.component('channel', {
    template: `
        <li class="channel active">
            <a class="channel_name" href="#">
                <span class="unread">0</span>
                <span><span class="prefix">#</span>general</span>
            </a>
        </li>
    `
})


Vue.component('channels', {
    template: `
        <div class="listings">
            <div class="listings_channels">
                <h2 class="listings_header">Channels</h2>
                <ul class="channel_list">
                    <channel></channel>
                </ul>
            </div>
            <div class="listings_direct-messages"></div>
        </div>
    `
})


Vue.component('message', {
    template: `
        <div class="message">
            <a href="#" class="message_profile-pic"></a>
            <a href="#" class="message_username">scotch</a>
            <span class="message_timestamp"></span>
            <span class="message_star"></span>
            <span class="message_content">Message</span>
        </div>
    `
})


Vue.component('messages', {
    template: `
        <div class="message-history">
            <message></message>
        </div>
    `
})


var app = new Vue({
    el: '#app'
})
```

#### Подгрузка временных сообщений

```python
...

class Channel(web.View):

    async def get(self):
        return web.json_response({
            'messages': [
                {'username': 'CIA Agent', 'text': "At least you can talk. Who are you?"},
                {'username': 'Bane', 'text': "It doesn't matter who we are... what matters is our plan."},
                {'username': 'Bane', 'text': "No one cared who I was until I put on the mask."},
            ]
        })
```


```js
var app = new Vue({
    el: '#app',

    data: {
        messages: [],
    },

    methods: {
        getChannelHistory: function() {
            axios.get('/general/history').then(function(response) {
                app.messages.unshift.apply(app.messages, response.data.messages);
            });
        }
    },

    created: function() {
        this.getChannelHistory();
    }
})
```

#### Отображение сообщений

```html
<div class="main">
    <channels></channels>
    <messages :messages="messages"></messages>
</div>
```

```js
Vue.component('message', {
    props: ['message'],

    template: `
        <div class="message">
            <a href="#" class="message_profile-pic"></a>
            <a href="#" class="message_username">{{ message.username }}</a>
            <span class="message_timestamp"></span>
            <span class="message_star"></span>
            <span class="message_content">{{ message.text }}</span>
        </div>
    `
})


Vue.component('messages', {
    props: ['messages'],

    template: `
        <div class="message-history">
            <message
                v-for="message in messages"
                :message="message">
            </message>
        </div>
    `
})
```

#### Добавление сообщениям идентификаторов

```python
class Channel(web.View):

    async def get(self):
        return web.json_response({
            'messages': [
                {'id': 0, 'username': 'CIA Agent', 'text': "At least you can talk. Who are you?"},
                {'id': 1, 'username': 'Bane', 'text': "It doesn't matter who we are... what matters is our plan."},
                {'id': 2, 'username': 'Bane', 'text': "No one cared who I was until I put on the mask."},
            ]
        })
```

```js
Vue.component('messages', {
    props: ['messages'],

    template: `
        <div class="message-history">
            <message
                v-for="message in messages"
                :message="message"
                :key="message.id">
            </message>
        </div>
    `
})
```

#### Шаблон для отправки сообщений

```js
Vue.component('app-footer', {
    template: `
        <div class="footer">
            <div class="user-menu">
                <span class="user-menu_profile-pic"></span>
                <span class="user-menu_username">scotch</span>
                <img class="connection_icon" src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAMAAABEpIrGAAABmFBMVEUAAAD////////////////////////////////////2+/LR5bKw1Hmfy1KUxz2VyD2izVKz1nnS5rP////A3JuOw0qKwkCNxD+QxT6Sxj6Txz6SxUnC3Jv1+fGXx2GDvkCGwECIwUCLwj+PxD6PxT+JwUCFwECZyGD2+vGSxWF9vEGAvkGDv0CMwz+Wx2GPw2F4ukJ7u0J+vUGBvkGHwUB8u0KSxGG31pp0uEN3uUJ5u0KFv0CCv0B6u0K415p5uU1yt0N/vUF1uEN8u0zG3bFttURwtkR5ukLH3rGWxnlqtERutUR2uUOZx3l6uVZos0VvtkRxt0Nzt0N8ulVisUVlskVns0VzuENmskVfsEVps0VztlZer0VhsEVjsUVstER1t1aOwXhcrkZdr0VgsEaQwnm/2a9YrUZbrka/2rDz+PFhr09XrEZksE6pzplUq0ZVrEZarUaqzpl0tWJRq0dWrEZ1tmJztWJOqUdSq0dxtGJMqEdNqUdQqkdytWKmzJhXrFBKqEdZrU+716+GvXhjr1dIp0hkr1dYtVOVAAAAFHRSTlMAV8/v/wCH+x/n////////////9kvBHZAAAAG7SURBVHgBvdOxjtNAEIDhGe/MZO3sxVaiIJkiSNdQUPJOeQlqXoCCIg/EU9BQHRKg5CT7ErzrHTa+aBOqaxC/tdLK+2kbj+H/hoWhlCmQr0HeyYxyM8mvkWHKoAfBS6cBWEeYugAzf4QGp1SV8DvU/ZjBdN7iud6hdnOTdl+TuALyrUPEwfdu3nc1ipr9AwdIFZPysJylRDfa6cZL2rfgMd9QjO8R0Y+/u7sa4LHZz4wN/MXEyw1hbK1VZdV7PZ1OyufzktsxXADCW5EkXq06Paan02Uoo3kHmAEzJ8HBN6v5qlkqaxTmCdAzQK8Noi6rXwCrJyutepUMAARnXS++3cvm2xvftR0PzAyQAXtwdNChifvFHppBdR003IDCIg6JDOse4DX8WIdo1TwfpaUgqWC9c4eqqg5HF20QZdAMmDlasdHWkrKR03J0A4iIXRTrpba29laiY8YMyOyMKYkXroyROZZuwVTyztAFJPmZKBGq+FxFVBr5BHr7ubd3GICfAM+88qDHHYe/BmbbIAaGKU/Fz10emDxyHxBhgJTg+DGP3O3QbltMBkd92F2H9sWxB772wo9z2z8FfwDHWbdKLDfq1AAAAABJRU5ErkJggg==">
                <span class="connection_status">online</span>
            </div>
            <div class="input-box">
                <input type="text" class="input-box_text" v-model="newMsg" @keypress="sendMessage">
            </div>
        </div>
    `,

    data: function() {
        return {
            newMsg: '',
        }
    },

    methods: {
        sendMessage(kbEvent) {
            if (kbEvent.keyCode == 13 && this.newMsg != '') {
                app.messages.push({
                    'username': 'anonymous',
                    'text': this.newMsg,
                    'id': 3
                })
                this.newMsg = '';
            }
        }
    }
})
```

#### Броадкаст сообщений

```python
import json
...

class WebSocketHandler(web.View):

    async def get(self):
        ws = web.WebSocketResponse()
        await ws.prepare(self.request)

        channel_waiters = self.request.app['waiters']['general']
        channel_waiters.append(ws)

        try:
            async for msg in ws:
                if msg.tp == web.MsgType.text:
                    data = json.loads(msg.data)
                    data['username'] = self.request.session['user']
                    channel_waiters.broadcast(json.dumps(data))
                elif msg.tp == web.MsgType.error:
                    print('connection closed with exception')
        finally:
            if ws in channel_waiters:
                channel_waiters.remove(ws)

        await ws.close()
        return ws
```


```python
...
from collections import defaultdict
...
from chat.views import Index, Channel, WebSocketHandler
...


class BroadcastList(list):

    def broadcast(self, message):
        for waiter in self:
            try:
                waiter.send_str(message)
            except Exception as exc:
                print('error was happining during broadcast', exc)


async def create_app():
    ...
    async def close_websockets(app):
        for channel in app['waiters'].values():
            while channel:
                ws = channel.pop()
                await ws.close(code=1000, message='Server shutdown')
    ...
    app['waiters'] = defaultdict(BroadcastList)
    ...
    app.router.add_get('/general/messages', WebSocketHandler)
    ...
    app.on_shutdown.append(close_websockets)
    return app
```


```js
sendMessage(kbEvent) {
    if (kbEvent.keyCode == 13 && this.newMsg != '') {
        app.ws.send(JSON.stringify({
            'text': this.newMsg,
        }));
        this.newMsg = '';
    }
}
```

```js
...
data: {
    messages: [],
    ws: null,
},

...
created: function() {
    var ws = new WebSocket('ws://' + window.location.host + '/general/messages');
    ws.onmessage = function(event) {
        var data = JSON.parse(event.data);
        if (data.hasOwnProperty('text')) {
            app.messages.push(data);
        }
    }
    this.ws = ws;
    
    this.getChannelHistory();
}
```

#### Добавляем каналы


#### Хранение сообщений в mongodb и подгрузка сообщений

```python
import asyncio
import motor.motor_asyncio
import faker
import random
from datetime import datetime as dt


client = motor.motor_asyncio.AsyncIOMotorClient('localhost', 27017)
db = client.test_database
collection = db.test_collection


def generate_collection_data(N):
    f = faker.Faker()
    channels = ('general', 'random')
    messages = ({
        'text': f.text(),
        'username': f.user_name(),
        'ts': dt.timestamp(f.date_time()),
        'channel': channels[random.choice([0,1])]
    } for _ in range(N))
    return messages


async def do_insert():
    messages = generate_collection_data(30)
    await collection.insert_many(messages)
    count = await collection.count()
    print(f"count: {count}")


loop = asyncio.get_event_loop()
loop.run_until_complete(do_insert())
```

#### Добавление регистрации и модели пользователя
