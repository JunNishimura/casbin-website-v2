---
title: Авторизация в APISIX с использованием Casbin
author: Rushikesh Tote
authorTitle: Член Касбина
authorURL: "http://github.com/rushitote"
authorImageURL: "https://avatars.githubusercontent.com/rushitote"
---

## Введение

[APISIX](https://apisix.apache.org/) это высокая производительность и масштабируемый облачный шлюз API, основанный на Nginx и т.д.. Это проект с открытым исходным кодом от Apache Software Foundation. Кроме того, что делает APISIX настолько хорошим является поддержка многих больших встроенных плагинов, которые могли бы быть использованы для реализации функций, таких как аутентификация, мониторинг, маршрутизация и т.д. И тот факт, что плагины в APISIX перезагружены (без перезагрузки) делает это очень динамичным.

Но при использовании APISIX могут быть сценарии, где вы можете добавить сложную логику авторизации в вашем приложении. Вот где authz-casbin может вам помочь, authz-casbin - это APISIX плагин, основанный на [Lua Casbin](https://github.com/casbin/lua-casbin/) , который обеспечивает мощную авторизацию на основе различных моделей управления доступом. [Casbin](/) is an authorization library which supports access control models like ACL, RBAC, ABAC. Первоначально написано в Го, он был портирован на многие языки, а Луа Касбин является Луа реализации Casbin. Разработка authz-casbin началась, когда мы предложили новый плагин для авторизации в хранилище APISIX ([#4674](https://github.com/apache/apisix/issues/4674)), с которым согласились основные участники. А после полезных отзывов, которые привели к значительным изменениям и улучшениям, PR ([#4710](https://github.com/apache/apisix/pull/4710)) наконец был объединен.

В этом блоге, мы будем использовать плагин authz-casbin для того, чтобы показать, как можно реализовать модель авторизации на основе контроля доступа на основе ролей (RBAC) в APISIX.

**ПРИМЕЧАНИЕ**: Вам нужно будет использовать некоторые другие плагины или пользовательский рабочий процесс для аутентификации пользователя, поскольку Касбин будет делать авторизацию только и не аутентификацию.

## Создание модели

Плагин использует три параметра для авторизации любого запроса - тема, объект и действие. Здесь тема является значением заголовка имени пользователя, который может быть что-то вроде `[username: alice]`. Затем объект - это путь к URL, к которому он доступен, и используется метод запроса запроса.

Скажем, мы хотим создать модель с тремя ресурсами на путях - `/`, `/res1` и `/res2`. И мы хотим иметь такую модель:

![изображение](https://i.imgur.com/7BlvBNR.png)

Это означает, что все пользователи (`*`), например `Джек` могут получить доступ к домашней странице (`/`). А пользователи с `правами администратора` , такими как `alice` и `bob` могут получить доступ ко всем страницам и ресурсам (например, `res1` и `res2`). Кроме того, давайте ограничим пользователей без прав администратора использовать только `GET` метод запроса. Для этого сценария мы можем определить модель:

```ini
[request_definition]
r = sub, obj, действовать

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = где-то (стр. ft == allow))

[matchers]
м = (g(r.sub, p.sub) || keyMatch(r.sub, p. ub)) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p.act)
```

## Создание политики

В соответствии с вышеупомянутым сценарием политика будет заключаться в следующем:

```csv
p, *, /, GET
p, admin, *, *
g, alice, admin
g, bob, admin
```

Матчер из модели означает:

1. `(g(r.sub, p.sub) || keyMatch(r.sub, p. ub))`: Тема запроса имеет роль в качестве темы политики или тема запроса совпадает с темой политики `keyMatch`. `keyMatch` построен в функции Lua Casbin, вы можете посмотреть описание функции и другие функции, которые могут быть полезны [здесь](https://github.com/casbin/lua-casbin/blob/master/src/util/BuiltInFunctions.lua).
2. `keyMatch(r.obj, p.obj)`: Объект запроса соответствует объекту политики (путь к URL здесь).
3. `keyMatch(r.act, p.act)`: Действие запроса соответствует действию политики (метод HTTP запроса здесь).

## Включение плагина на маршрут

Создав модель и политику, вы можете включить ее на маршруте с помощью APISIX Admin API. Чтобы включить его, используя пути файлов моделей и политики:

```sh
curl http://127.0.0. :9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "authz-casbin": {
            "model_path": "/path/to/model. onf",
            "policy_path": "/path/to/policy. sv",
            "username": "username"
        }
    },
    "upstream": {
        "nodes": {
            "127. .0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "uri": "/*"
}'
```

Здесь имя пользователя - это имя заголовка, которое вы будете использовать для передачи в теме. Например, если вы будете передавать заголовок имени пользователя как `пользователя: alice`, вы бы использовали `"username": "user"`.

Для использования текста модели/политики вместо файлов можно использовать `модель` и `политику` поля:

```sh
curl http://127.0.0. :9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "authz-casbin": {
            "model": "[request_definition]
            r = sub, obj, act

            [policy_definition]
            p = sub, obj, действовать

            [role_definition]
            g = _, _

            [policy_effect]
            e = где-то (стр. f== допустить))

            [matchers]
            м = (g(r. ub, p. ub) || keyMatch(r.sub, p.sub)) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p. ct)",

            "policy": "p, *, /, GET
            p, admin, *, *
            g, угла, администратора
            g, bob, администратор",

            "Имя пользователя": "Имя пользователя"
        }
    },
    "upstream": {
        "nodes": {
            "127. .0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "uri": "/*"
}'
```

## Включение плагина с использованием глобальной модели/политики

Могут возникнуть ситуации, когда вы можете захотеть использовать одну модель и конфигурацию политики для нескольких маршрутов. Вы можете сделать это, отправив запрос `PUT` для добавления модели и конфигурации политики к метаданным плагина:

```sh
curl http://127.0.0. :9080/apisix/admin/plugin_metadata/authz-casbin -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -i -X PUT -d '
{
"model": "[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = где-то (стр. ft == allow))

[matchers]
м = (g(r.sub, p.sub) || keyMatch(r. ub, p.sub)) && keyMatch(r.obj, p.obj) && keyMatch(r. ct, p.act)",

"policy": "p, *, /, GET
p, admin, *, *
g, alice, admin
g, bob, admin"
}'
```

Затем чтобы включить одну и ту же конфигурацию на некоторых маршрутах, отправьте запрос, используя админ-API:

```sh
curl http://127.0.0. :9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "authz-casbin": {
            "username": "username"
        }
    },
    "upstream": {
        "nodes": {
            "127. .0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "uri": "/route1/*"
}'
```

Это добавит конфигурацию метаданных плагина в маршрут. Вы также можете легко обновить конфигурацию метаданных плагина, переслав запрос к метаданным плагина с обновленной моделью и конфигурацией политики, плагин автоматически обновит все маршруты, используя метаданные плагина.

## Варианты использования

- Основным вариантом использования этого плагина будет внедрение авторизации в ваших API. Вы можете легко добавить этот плагин на любой API маршрут, который используется с вашей моделью авторизации и конфигурацией политики.
- Если вы хотите иметь одну модель авторизации для всех ваших API, вы можете использовать глобальный метод модели/политики. Это делает обновление политики простыми для всех маршрутов, так как вам нужно обновить только метаданные в и т.д.
- Если вы хотите использовать другую модель для каждого маршрута, вы можете использовать метод маршрута. Это полезно, когда различные маршруты API имеют различные наборы прав пользователей. Вы также можете использовать это при работе с большими правилами, так как это сделает авторизацию быстрее, если отфильтровать по нескольким маршрутам.