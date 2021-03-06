# polls

[参考](https://github.com/aio-libs/aiohttp-demos/tree/master/demos/blog)

## 准备阶段

- 项目结构

```
polls  # 项目根文件夹
├── aiohttpdemo_polls  # 应用文件夹
│   ├── static  # 静态文件
│   │   ├── images
│   │   │   └── background.png
│   │   └── style.css
│   ├── templates  # 模版文件
│   │   ├── 404.html
│   │   ├── 500.html
│   │   ├── base.html
│   │   ├── detail.html
│   │   ├── index.html
│   │   └── results.html
│   ├── db.py  # 数据库类和函数
│   ├── main.py  # web服务开始文件
│   ├── middlewares.py  # 中间件
│   ├── routes.py  # 路由
│   ├── settings.py  # 配置文件
│	├── utilis.py # 命令行配置文件
│   └── views.py  # 视图
├── config
│   └── polls.yaml  # 数据库配置
├── init_db.py  # 初始化构建数据库表和数据
```

- 环境

运行环境：python=3.7.*

```
pip install aiohttp
pip install aiomysql
pip install pyyaml
pip install aiohttp_jinja2
```

## 视图

```python
# views.py
import aiohttp_jinja2
from aiohttp import web

import db


@aiohttp_jinja2.template('index.html')
async def index(request):
    async with request.app['db'].acquire() as conn:
        cursor = await conn.execute(db.question.select())
        records = await cursor.fetchall()
        questions = [dict(q) for q in records]
        return {'questions': questions}


@aiohttp_jinja2.template('detail.html')
async def poll(request):
    async with request.app['db'].acquire() as conn:
        question_id = request.match_info['question_id']
        try:
            question, choices = await db.get_question(conn,
                                                      question_id)
        except db.RecordNotFound as e:
            raise web.HTTPNotFound(text=str(e))
        return {
            'question': question,
            'choices': choices
        }


@aiohttp_jinja2.template('results.html')
async def results(request):
    async with request.app['db'].acquire() as conn:
        question_id = request.match_info['question_id']

        try:
            question, choices = await db.get_question(conn,
                                                      question_id)
        except db.RecordNotFound as e:
            raise web.HTTPNotFound(text=str(e))

        return {
            'question': question,
            'choices': choices
        }


async def vote(request):
    async with request.app['db'].acquire() as conn:
        question_id = int(request.match_info['question_id'])
        data = await request.post()
        try:
            choice_id = int(data['choice'])
        except (KeyError, TypeError, ValueError) as e:
            raise web.HTTPBadRequest(
                text='You have not specified choice value') from e
        try:
            await db.vote(conn, question_id, choice_id)
        except db.RecordNotFound as e:
            raise web.HTTPNotFound(text=str(e))
        router = request.app.router
        url = router['results'].url_for(question_id=str(question_id))
        return web.HTTPFound(location=url)

```

## 项目配置文件

直接读取版

```python
import pathlib
import yaml


BASE_DIR = pathlib.Path(__file__).parent.parent
config_path = BASE_DIR/'config'/'polls.yaml'


def get_config(path):
    with open(path) as f:
        config = yaml.safe_load(f)
    return config

config = get_config(config_path)

```

命令行读取版

```python
import argparse
import pathlib

from trafaret_config import commandline

from aiohttpdemo_polls.utils import TRAFARET


BASE_DIR = pathlib.Path(__file__).parent.parent
DEFAULT_CONFIG_PATH = BASE_DIR / 'config' / 'polls.yaml'


def get_config(argv=None):
    ap = argparse.ArgumentParser()
    commandline.standard_argparse_options(
        ap,
        default_config=DEFAULT_CONFIG_PATH
    )

    # ignore unknown options
    options, unknown = ap.parse_known_args(argv)

    config = commandline.config_from_options(options, TRAFARET)
    return config
```

## 数据库

配置

```yaml
mysql:
  database: polls
  user: root
  password: ""
  host: localhost
  port: 3306
```

初始化

```python
from sqlalchemy import MetaData, create_engine
from aiohttpdemo_polls.settings import config
from aiohttpdemo_polls.db import question, choice

DSN = "mysql+pymysql://{user}:{password}@{host}:{port}/{database}"

def create_tables(engine):
    meta = MetaData()
    meta.create_all(bind=engine, tables=[question, choice])


def sample_data(engine):
    conn = engine.connect()
    conn.execute(question.insert(), [
        {'question_text': 'What\'s new?',
         'pub_date': '2015-12-15 17:17:49'}
    ])
    conn.execute(choice.insert(), [
        {'choice_text': 'Not much', 'votes': 0, 'question_id': 1},
        {'choice_text': 'The sky', 'votes': 0, 'question_id': 1},
        {'choice_text': 'Just hacking again', 'votes': 0, 'question_id': 1},
    ])
    conn.close()


if __name__ == '__main__':
    db_url = DSN.format(**config['mysql'])
    engine = create_engine(db_url)
    create_tables(engine)
    sample_data(engine)
```

模型文件

```python
from sqlalchemy  import  MetaData, Table, Column, ForeignKey
from sqlalchemy import Integer, String, DateTime

from aiomysql.sa import create_engine

__all__ = ['question', 'choice']


meta = MetaData()

question = Table(
    'question', meta,

    Column('id', Integer, primary_key=True),
    Column('question_text', String(200), nullable=False),
    Column('pub_date', DateTime, nullable=False)
)

choice = Table(
    'choice', meta,

    Column('id', Integer, primary_key=True),
    Column('choice_text', String(200), nullable=False),
    Column('votes', Integer, server_default="0", nullable=False),

    Column('question_id',
           Integer,
           ForeignKey('question.id', ondelete='CASCADE'))
)


async def init_mysql(app):
    conf = app['config']['mysql']
    engine= await create_engine(
        db=conf['database'],
        user=conf['user'],
        password=conf['password'],
        host=conf['host'],
        port=conf['port']
    )
    app['db'] = engine

class RecordNotFound(Exception):
    """Requested record in database was not found"""

async def close_mysql(app):
    app['db'].close()
    await app['db'].wait_closed()


async def get_question(conn, question_id):
    result = await conn.execute(
        question.select()
        .where(question.c.id == question_id))
    question_record = await result.first()
    if not question_record:
        msg = "Question with id: {} does not exists"
        raise RecordNotFound(msg.format(question_id))
    result = await conn.execute(
        choice.select()
        .where(choice.c.question_id == question_id)
        .order_by(choice.c.id))
    choice_records = await result.fetchall()
    return question_record, choice_records


async def vote(conn, question_id, choice_id):
    await conn.execute(
        choice.update()
        .returning(*choice.c)  # 报错
        .where(choice.c.question_id == question_id)
        .where(choice.c.id == choice_id)
        .values(votes=choice.c.votes+1))
    record = await result.fetchone()
    if not record:
        msg = "Question with id: {} or choice id: {} does not exists"
        raise RecordNotFound(msg.format(question_id, choice_id))
```

## 模版

404

```
{% extends "base.html" %}

{% set title = "Page Not Found" %}
```

500

```
{% extends "base.html" %}

{% set title = "Internal Server Error" %}

```

base

```
<!DOCTYPE html>
<html>
  <head>
    {% block head %}
        <link rel="stylesheet" type="text/css"
              href="{{ url('static', filename='style.css') }}" />
        <title>{{title}}</title>
    {% endblock %}
  </head>
  <body>
    <h1>{{title}}</h1>
    <div>
      {% block content %}
      {% endblock %}
    </div>
  </body>
</html>

```

detail

```
{% extends "base.html" %}

{% set title = question.question_text %}

{% block content %}
{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{{ url('vote', question_id=question.id) }}" method="post">
{% for choice in choices %}
    <input type="radio" name="choice" id="choice{{ loop.index }}" value="{{ choice.id }}" />
    <label for="choice{{ loop.index }}">{{ choice.choice_text }}</label><br />
{% endfor %}
<input type="submit" value="Vote" />
</form>
{% endblock %}

```

index

```
{% extends "base.html" %}

{% set title = "Main" %}

{% block content %}
{% if questions %}
    <ul>
    {% for question in questions %}
    <li><a href="{{ url('poll', question_id=question.id) }}">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}
{% endblock %}

```

results

```
{% extends "base.html" %}

{% set title = question.question_text %}

{% block content %}
<ul>
{% for choice in choices %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote(s)</li>
{% endfor %}
</ul>

<a href="{{ url('poll', question_id=question.id) }}">Vote again?</a>
{% endblock %}
```

## 静态文件

style.css

```css
li a {
    color: green;
}

body {
    background: white url("images/background.png") no-repeat;
}
```

## 中间件

```python
import aiohttp_jinja2
from aiohttp import web


async def handle_404(request):
    return aiohttp_jinja2.render_template('404.html', request, {})


async def handle_500(request):
    return aiohttp_jinja2.render_template('500.html', request, {})


def create_error_middleware(overrides):

    @web.middleware
    async def error_middleware(request, handler):

        try:
            response = await handler(request)

            override = overrides.get(response.status)
            if override:
                return await override(request)

            return response

        except web.HTTPException as ex:
            override = overrides.get(ex.status)
            if override:
                return await override(request)

            raise

    return error_middleware


def setup_middlewares(app):
    error_middleware = create_error_middleware({
        404: handle_404,
        500: handle_500
    })
    app.middlewares.append(error_middleware)
```

## 命令行配置文件

```python
# utils.py
import trafaret as T


TRAFARET = T.Dict({
    T.Key('postgres'):
        T.Dict({
            'database': T.String(),
            'user': T.String(),
            'password': T.String(),
            'host': T.String(),
            'port': T.Int(),
            'minsize': T.Int(),
            'maxsize': T.Int(),
        }),
    T.Key('host'): T.IP,
    T.Key('port'): T.Int(),
})
```

## 主文件

```python
from aiohttp import web
from routes import setup_routes
from settings import config, BASE_DIR
from db import close_mysql, init_mysql
import  aiohttp_jinja2, jinja2
from middlewares import setup_middlewares

# 创建应用
app  = web.Application()
# 加载配置
app['config'] = config
# 添加模版文件
aiohttp_jinja2.setup(app, loader=jinja2.FileSystemLoader(str(BASE_DIR/'aiohttpdemo_polls'/'templates')))
# 添加路由
setup_routes(app)
# 添加中间件
setup_middlewares(app)
# 启动时初始化数据库
app.on_startup.append(init_mysql)
# 结束时关闭数据库
app.on_cleanup.append(close_mysql)
# 运行应用
web.run_app(app=app)
```

命令行启动相应的主文件

```python
import logging
import sys

import aiohttp_jinja2
import jinja2
from aiohttp import web

from aiohttpdemo_polls.db import close_pg, init_pg
from aiohttpdemo_polls.middlewares import setup_middlewares
from aiohttpdemo_polls.routes import setup_routes
from aiohttpdemo_polls.settings import get_config


async def init_app(argv=None):

    app = web.Application()

    app['config'] = get_config(argv)

    # setup Jinja2 template renderer
    aiohttp_jinja2.setup(
        app, loader=jinja2.PackageLoader('aiohttpdemo_polls', 'templates'))

    # create db connection on startup, shutdown on exit
    app.on_startup.append(init_pg)
    app.on_cleanup.append(close_pg)

    # setup views and routes
    setup_routes(app)

    setup_middlewares(app)

    return app


def main(argv):
    logging.basicConfig(level=logging.DEBUG)

    app = init_app(argv)

    config = get_config(argv)
    web.run_app(app,
                host=config['host'],
                port=config['port'])


if __name__ == '__main__':
    main(sys.argv[1:])
```

命令行

```
python  
```

## 