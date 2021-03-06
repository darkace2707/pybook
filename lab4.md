# Лабораторная работа №4. Работа с API ВКонтакте


Эта работа посвящена взаимодействию с [API ВКонтакте](https://vk.com/dev/manuals).

<div class="alert alert-info">
<b>Замечание</b>: Если вы не знакомы с термином "API", то рекомендую прочитать статью: <a href="https://medium.freecodecamp.org/what-is-an-api-in-english-please-b880a3214a82">What is an API? In English, please.</a>
</div>

Чтобы начать работать с API от вас требуется зарегистрировать новое приложение. Для этого зайдите на форму создания нового Standalone приложения [https://vk.com/editapp?act=create](https://vk.com/editapp?act=create) и следуйте инструкциям. Вашему приложению будет назначен идентификатор, который потребуется для выполнения работы.

Запросы к API ВКонтакте имеют следующий формат ([из документации](https://vk.com/dev/api_requests)):
```
https://api.vk.com/method/METHOD_NAME?PARAMETERS&access_token=ACCESS_TOKEN&v=V
```

где:
* `METHOD_NAME` - это название метода API, к которому Вы хотите обратиться.
* `PARAMETERS` - входные параметры соответствующего метода API, последовательность пар `name=value`, объединенных амперсандом `&`.
* `ACCESSS_TOKEN` - ключ доступа.
* `V` - используемая версия API (в настоящий момент 5.53).


Например, чтобы получить список друзей, с указанием их пола, нужно выполнить следующий запрос:
```
https://api.vk.com/method/friends.get?fields=sex&access_token=0394a2ede332c9a13eb82e9b24631604c31df978b4e2f0fbd2c549944f9d79a5bc866455623bd560732ab&v=5.53
```

<div class="alert alert-danger">
<strong>Внимание:</strong> Токен доступа ненастоящий, поэтому этот запрос работать не будет.
</div>

Чтобы получить токен доступа вы можете воспользоваться написанным для вас скриптом `access_token.py` следующим образом:

```sh
$ python access_token.py YOUR_CLIENT_ID -s friends,messages
```

где вместо `YOUR_CLIENT_ID` необходимо подставить идентификатор вашего приложения.

После выполнения команды откроется новая вкладка браузера, из адресной строки которого вам необходимо скопировать токен доступа.

![](assets/access_token.png)

<div class="alert alert-info">
<strong>Внимание:</strong> На этом этапе вы можете повторить ранее представленный пример запроса, чтобы убедиться, что вы делаете все верно.
</div>

Далее приведено содержимое файла `access_token.py`:
```python
import webbrowser
import argparse


def get_access_token(client_id, scope):
    assert isinstance(client_id, int), 'clinet_id must be positive integer'
    assert isinstance(scope, str), 'scope must be string'
    assert client_id > 0, 'clinet_id must be positive integer'
    url = """\
    https://oauth.vk.com/authorize?client_id={client_id}&\
    redirect_uri=https://oauth.vk.com/blank.hmtl&\
    scope={scope}&\
    &response_type=token&\
    display=page\
    """.replace(" ", "").format(client_id=client_id, scope=scope)
    webbrowser.open_new_tab(url)


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("client_id", help="Application Id", type=int)
    parser.add_argument("-s",
                        dest="scope",
                        help="Permissions bit mask",
                        type=str,
                        default="",
                        required=False)
    args = parser.parse_args()
    get_access_token(args.client_id, args.scope)

```

**Задание №1**. Требуется написать функцию прогнозирования возраста пользователя по возрасту его друзей. 

```python
def get_friends(user_id, fields):
    """ Returns a list of user IDs or detailed information about a user's friends """
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert isinstance(fields, str), "fields must be string"
    assert user_id > 0, "user_id must be positive integer"
    # PUT YOUR CODE HERE
    pass
```

Для выполнения этого задания нужно получить список всех друзей для указанного пользователя, отфильтровать тех у кого возраст не указан или указаны только день и месяц рождения.

Для выполнения запросов к API мы будем использовать библиотеку [`requests`](http://docs.python-requests.org/en/master/):

```sh
$ pip install requests
```

Список пользователей можно получить с помощью метода [`friends.get`](https://vk.com/dev/friends.get). Ниже приведен пример обращения к этому методу для получения списка всех друзей указанного пользователя:

```python
domain = "https://api.vk.com/method"
access_token = # PUT YOUR ACCESS TOKEN HERE
user_id = # PUT USER ID HERE

query_params = {
    'domain' : domain,
    'access_token': access_token,
    'user_id': user_id,
    'fields': 'sex'
}

query = "{domain}/friends.get?access_token={access_token}&user_id={user_id}&fields={fields}&v=5.53".format(**query_params)
response = requests.get(query)
```

Функция `requests.get` выполняет [GET запрос](https://ru.wikipedia.org/wiki/HTTP#GET) и возвращает объект `Response`, который представляет собой ответ сервера на посланный нами запрос.

Объект `Response` имеет множество атрибутов:
```python
>>> response.<tab>
response.apparent_encoding      response.history                response.raise_for_status
response.close                  response.is_permanent_redirect  response.raw
response.connection             response.is_redirect            response.reason
response.content                response.iter_content           response.request
response.cookies                response.iter_lines             response.status_code
response.elapsed                response.json                   response.text
response.encoding               response.links                  response.url
response.headers                response.ok
```

Нас будет интересовать только метод `response.json`, который возвращает [JSON](https://ru.wikipedia.org/wiki/JSON) объект:

```python
>>> response.json()
{'response': {'count': 136,
              'items': [{'first_name': 'Drake',
                         'id': 1234567,
                         'last_name': 'Wayne',
                         'online': 0,
                         'sex': 1},
                        {'first_name': 'Gracie'
                         'id': 7654321,
                         'last_name': 'Habbard',
                         'online': 0,
                         'sex': 0},
                         ...
>>> response.json()['response']['count']
136
>>> response.json()['response']['items'][0]['first_name']
'Drake'
```

Поле `count` содержит число записей, а `items` список словарей с информацией по каждому пользователю.

Выполняя запросы мы не можем быть уверены, что не возникнет ошибок. Возможны различные ситуации, например:
- есть неполадки в сети;
- удаленный сервер по какой-то причине не может обработать запрос;
- мы слишком долго ждем ответ от сервера.

В таких случаях необходимо попробовать повторить запрос. При этом повторные запросы желательно посылать не через константные промежутки времени, а по алгоритму экспоненциальной задержки.

<div class="alert alert-info">
Описание алгоритма с примерами можно найти в статье <a href="https://habrahabr.ru/post/227225/">Exponential Backoff или как «не завалить сервер»</a>. Почитать про обработку исключений при работе с библиотекой <tt>requests</tt> можно <a href="https://khashtamov.com/ru/python-requests/">тут</a>.
</div>

Напишите функцию `get()`, которая будет выполнять GET-запрос к указанному адресу, а при необходимости повторять запрос указанное число раз по алгоритму экспоненциальной задержки:

```python
def get(url, params={}, timeout=5, backoff_factor=0.3):
    """ Sends a GET request """
    pass
```

```python
>>> get("https://httpbin.org/get")
>>> <Response [200]>

>>> get("https://httpbin.org/delay/2", timeout=1)
ReadTimeout: HTTPSConnectionPool(host='httpbin.org', port=443): Read timed out. (read timeout=1)

>>> get("https://httpbin.org/status/500")
HTTPError: 500 Server Error: INTERNAL SERVER ERROR for url: https://httpbin.org/status/500

>>> get("https://noname.com", timeout=1)
ConnectionError: HTTPSConnectionPool(host='noname.com', port=443): Max retries exceeded with url: /
```

На текущий момент вы должны заполнить тело функции `get_friends` так, чтобы она возвращала список друзей для указанного пользователя. Аргумент `fields` представляет из себя строку, в которой через запятую указываются какие поля необходимо получить по каждому пользователю.

Теперь мы можем написать функцию `age_predict` для "наивного" прогнозирования возраста пользователя с идентификатором `user_id`:

```python
def age_predict(user_id):
    """
    >>> age_predict(???)
    ???
    """
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert user_id > 0, "user_id must be positive integer"
    # PUT YOUR CODE HERE
    pass
```

<div class="alert alert-info">
<strong>Замечание:</strong> Для реализации этой функции вы можете использовать конструкцию <code>try...except</code>, где <code>except</code> будет содержать только <code>pass</code>.
</div>

**Задание №2:** Требуется написать функцию, которая бы выводила график частоты переписки с указанным пользователем.

Давайте начнем с того, что получим всю или часть переписки с указанным пользователем. Для этого вам потребуется реализовать метод API [messages.getHistory](https://vk.com/dev/messages.getHistory) по аналогии с тем, как вы реализовывали получение списка друзей пользователя:

```python
def messages_get_history(user_id, offset=0, count=20):
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert user_id > 0, "user_id must be positive integer"
    assert isinstance(offset, int), "offset must be positive integer"
    assert offset >= 0, "user_id must be positive integer"
    assert count >= 0, "user_id must be positive integer"
    # PUT YOUR CODE HERE
    pass
```

<div class="alert alert-danger">
<strong>Замечание:</strong> В реализации функции <code>messages_get_history</code> нужно учитывать, что API ВКонтакте устанавливает ограничение на число запросов в одну секунду (на сегодняшний день это три запроса). Число сообщений, которое вы можете получить за одно запрос - 200.
</div>

Далее приведен пример использования функции `messages_get_history`:
```python
>>> user_id = # PUT USER ID HERE
>>> history = messages_get_history(user_id)
>>> from pprint import pprint as pp
>>> message = history['response']['items'][0]
>>> pp(message)
{'body': 'Это текст сообщения.',
 'date': 1474811631,
 'from_id': USER_ID_HERE,
 'id': 168989,
 'out': 0,
 'read_state': 1,
 'user_id': USER_ID_HERE}
```

Каждое сообщение содержит следующие поля:
* `body` - текст сообщения;
* `date` - дата отправки сообщения в формате [unixtime](https://ru.wikipedia.org/wiki/UNIX-время);
* `from_id` - идентификатор автора сообщения;
* `id` - идентификатор сообщения;
* `out` - тип сообщения (0 — полученное, 1 — отправленное, не возвращается для пересланных сообщений);
* `read_state` - статус сообщения (0 — не прочитано, 1 — прочитано, не возвращается для пересланных сообщений);
* `user_id` - идентификатор пользователя, в диалоге с которым находится сообщение.

Нас интересует поле `date`, которое представляет дату отправки сообщения в формате unixtime. Чтобы изменить формат даты представления можно воспользоваться функцией  `strftime` из модуля `datetime`:

```python
>>> from datetime import datetime
>>> date = datetime.fromtimestamp(message['date']).strftime("%Y-%m-%d")
'2016-09-25'
```

Формат представления указывается в виде [строки форматирования](https://docs.python.org/2/library/datetime.html#strftime-strptime-behavior), например, `%Y-%m-%d` - год, месяц и день, соответственно.

На данный момент вашей задачей является написать функцию `count_dates_from_messages`, которая возвращает два списка: список дат и список частоты каждой даты, а принимает список сообщений:

```python
def count_dates_from_messages(messages):
    #PUT YOUR CODE HERE
    pass
```

<div class="alert alert-info">
<strong>Замечание:</strong> При большом количестве сообщений вы их можете разбить по неделям или месяцам.
</div>

Далее мы воспользуемся сервисом [Plot.ly](https://plot.ly/), который предоставляет API для рисования графиков. Запрос содержит информацию о точках, которые нужно отобразить на графике. Вам нужно зарегистрироваться и получить ключ доступа (`API KEY`). Для более простого взаимодействия с Plot.ly мы воспользуемся готовым модулем (существуют решения и для других языков):

```sh
$ pip3 install plotly
```

Перед началом его использования нужно провести предварительную [настройку](https://plot.ly/python/getting-started/), указав ключ доступа и имя пользователя:

```python
import plotly
plotly.tools.set_credentials_file(username='YOUR_USER_NAME', api_key='YOUR_API_KEY')
```

Ниже приведен пример построения графика, где переменная `x` содержит даты, а `y` - количество сообщений в этот день:

```python
import plotly.plotly as py
import plotly.graph_objs as go

from datetime import datetime
x = [datetime(year=2016, month=09, day=23),
     datetime(year=2016, month=09, day=24),
     datetime(year=2016, month=09, day=25)]

data = [go.Scatter(x=x,y=[142, 50, 8])]
py.iplot(data)
```

Созданный график вы можете найти в своем [профиле](https://plot.ly/organize/home):

![](assets/plotly.png)

**Задание №3**:

```python
def get_network(users_ids, as_edgelist=True):
    """ Building a friend graph for an arbitrary list of users """
    pass
```

```bash
$ pip install python-igraph
$ pip install numpy
$ pip install cairocffi
$ brew install cairo
```

```python
from igraph import Graph, plot
import numpy as np

vertices = [i for i in range(7)]
edges = [
    (0,2),(0,1),(0,3),
    (1,0),(1,2),(1,3),
    (2,0),(2,1),(2,3),(2,4),
    (3,0),(3,1),(3,2),
    (4,5),(4,6),
    (5,4),(5,6),
    (6,4),(6,5)
]

g = Graph(vertex_attrs={"label":vertices},
    edges=edges, directed=False)
g.simplify(multiple=True, loops=True)

communities = g.community_edge_betweenness(directed=False)
clusters = communities.as_clustering()

N = len(vertices)
visual_style["layout"] = g.layout_fruchterman_reingold(
    maxiter=1000,
    area=N**3,
    repulserad=N**3)

plot(g, **visual_style)
```
