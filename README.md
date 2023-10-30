# Описание протокола бронирования

Для автоматизации бронирования необходимо реализовать данный протокол с вашей стороны. Важно отметить, что наш сервер выполняет роль клиента и инициирует отправку запросов.

Аутентификация происходит с использованием API токена, который всегда передается в виде GET параметра ``api_key``.<br />
Формат даты и времени -  RFC3339 (2023-08-15T15:52:01+00:00).<br />
Все ответы, получаемые от сервера, должны быть в формате JSON.<br />

Термины, которые встречаются ниже:
Программа, program - Например, ``Ужин на теплоходе``<br />
Рейс, event - конкертный рейс на программу, в определённую дату и время. Например, ``Ужин на теплоходе, 10.06.2023 в 19:00``

[1. Расписание на день](#1-расписания-на-день)<br />
[2. Ближайшие рейсы](#2-ближайшие-рейсы)<br />
[3. Создание заказа](#3-создание-заказа)<br />
[4. Проверка возможности оплаты заказа](#4-проверка-возможности-оплаты-заказа)<br />
[5. Подтверждение заказа](#5-подтверждение-заказа)<br />
[6. Отмена заказа](#6-отмена-заказа)<br />
[7. Информация о заказе](#7-информация-о-заказе)<br />

## 1. Расписания на день

Запрос должен вернуть доступное расписание на запрошенный день, вместе с остатком мест. В ответе должны быть **только активные рейсы**, на которые возможна покупка.

Endpoint: GET `/get_schedule`

Параметры запроса:
 - **date** - дата в формате YYYY-MM-DD (Год-месяц-день)
 - **program_id** - ID программы в вашей системе 
 - **pier_id** - ID точки отправления в вашей системе. Параметр будет передан в том случае, если в вашей системе есть точки отправления.

Пример запроса:
``/get_schedule?date=2023-08-15&program_id=123&api_key=12345``

Формат ответа:
```JSON
{
    "success": true,
    "schedule": [
        {
            "time": "2023-08-15T15:52:01+00:00",
            "places_available": 10
        },
        {
            "time": "2023-08-15T15:52:01+00:00",
            "places_available": 0
        }
    ]
}
```

- **time** - дата и время рейса RFC3339
- **places_available** - Количество доступных мест

##### Зачем нужно передавать информацию об остатке мест?
Предположим, что фактически остаётся 3 свободных места, при этом, остаток мест нам не известен и соответственно мы не можем сообщить никак покупателю, что остаётся всего лишь 3 места. Тогда возникает ситуация, когда пользователь попытается купить, например, 5 билетов и это приведёт к ошибке. Почему бы тогда ему сразу не показать, сколько мест остаётся, что бы он спокойно выбрал тот рейс, где будут места для него?

## 2. Ближайшие рейсы

Запрос должен вернуть все рейсы в ближайший день, где доступен хоть один рейс, при этом отсчёт начинается с указанной в параметрах даты. Так же, билеты на эти рейсы должны быть доступны для покупки.

В качестве примера возьмём следующее расписание:
14.10.2023 - рейсов нет
15.10.2023 - 13:00, 14:00
16.10.2023 - 10:00, 11:00
17.10.2023 - 10:00, 11:00

При запросе с датой 14.10.2023, ответ должен содержать расписание на день 15.10.2023

Endpoint: GET ``/next_events``

Параметры запроса:
 - **date** - дата в формате YYYY-MM-DD (год-месяц-день)
 - **program_id** - ID программы в вашей системе 

Формат ответа:
```JSON
{
    "success": true,
    "programs": [
        "2023-10-15T13:00:00+03:00",
        "2023-10-15T14:00:00+03:00"
    ]
}
```

## 3. Создание заказа

Покупка проходит в 3 этапа:

1. Создание заказа происходит **перед** оплатой. Так как во время оплаты места могут неожиданно закончится, необходимо зарезервировать места за ним на 15 минут. 15 минут в данном случае, взято не просто так, ведь пользователь оплачивает заказ не моментально, ему нужно ввести данные своей карты, а скорее всего, перед этим её ещё надо и найти. Среднее время, за которое пользователь оплачивает заказ - 10 минут.

2. Непосредственно перед списанием средств у пользователя происходит проверка актуальности заказа, ведь отведённое время на бронь могло истечь или продажа была закрыта.

3. Уведомление об оплате заказа

endpoint: POST ``/order/create``

Параметры этого запроса передаются в JSON, в теле POST запроса, при этом в GET параметре передаётся только API токен

GET параметры запроса: 
- **api_key** - API токен

Тело запроса: 
- **our_id** - ID заказа в нашей системе
- **date** - дата рейса в формате YYYY-MM-DD
- **time** - время рейса в формате HH:mm (часы, минуты с ведущими нолями)
- **program_id** - id программы
- **tickets** - массив с билетами

Пример запроса: 
```JSON
{
    "our_id": 1,
    "date": "2077-01-01",
    "time": "10:00",
    "program_id": "123",
    "tickets": [
        {
            "type_id": "123",
            "quantity": 1,
        },
        {
            "type_id": "111",
            "quantity": 3
        }
    ]
}
```

Пример ответа: 
```JSON
{
    "success": true,
    "order": {
        "id": 1234,
        "tickets": [
            {
                "type_id": "123",
                "qr_code": "1234567890",
                "quantity": 1,
                "place": "2B",
            },
            {
                "type_id": "111",
                "qr_code": "1234567890",
                "place": "2A",
                "quantity": 3
            }
        ]
    }
}
```

- **order.id** - ID заказа в вашей системе
- **order.tickets.** - Билеты в заказе
- **order.tickets[].type_id** - id типа билета
- **order.tickets[].qr_code** - Контент QR кода, если есть. Необязательный параметр
- **order.tickets[].place** - Номер места, строка. Необязательный параметр
- **order.tickets[].quantity** - количество билетов в заказе


Пример ответа в случае ошибки:
```JSON
{
    "success": false,
    "message": "человекопонятное текстовое описание ошибки"
}
```

Описание ошибки не будет показано пользователю, но будет показано сотрудникам, которые работают с заказами.

## 4. Проверка возможности оплаты заказа

endpoint: GET ```/order/check```

Параметры:
- **order_id** - ID заказа

Пример ответа: 
```JSON
{
    "can_pay": true
}
```

- **can_pay** - Можно ли оплатить заказ. Если false, то пользователю будет показана ошибка и заказ оплатить не получится

## 5. Подтверждение заказа

endpoint: POST ``/order/confirm``

Параметры:
- **order_id** - ID заказа

Пример ответа: 
```JSON
{
    "success": true
}
```

## 6. Отмена заказа

Запрос должен отменять заказ в вашей системе.

endpoint: POST ``/order/cancel``

Параметры:
- **order_id** - ID заказа

Пример ответа в случае успеха: 
```JSON
{
    "success": true
}
```

Пример ответа в случае ошибки:
```JSON
{
    "success": false,
    "message": "человекопонятное текстовое описание ошибки"
}
```

## 7. Информация о заказе

Иногда, нужно смотреть, что с заказом уже после оплаты. Возможно, его перенесли, отменили или поменяли состав заказа. Для этого нужен этот запрос, который возвращает актуальную информацию о заказе.

endpoint: GET ``/order/info``

GET Параметры:
- **order_id** - ID заказа

Пример ответа в случае успеха: 
```JSON
{
    "success": true,
    "order": {
        "id": 123,
        "datetime": "2023-08-15T15:52:01+00:00",
        "status": 0,
        "program": {
            "id": 123,
            "name": "Ужин на теплоходе"
        },
        "pier": {
            "id": 1,
            "name": "Адмиралтейская набережная, 2"
        },
        "tickets": [
            {
                "type_id": "123",
                "qr_code": "1234567890",
                "quantity": 1,
                "place": "2B",
            },
            {
                "type_id": "111",
                "qr_code": "1234567890",
                "place": "2A",
                "quantity": 3
            }
        ]
    }
}
```

- **order.id** - ID заказа
- **order.datetime** - Дата и время поездки
- **order.status** - Статус заказа. Возможные значения: 0 - не оплачен, 1 - оплачен, 2 - отменён.
- **order.program.id** - ID программы
- **order.program.name** - Название  программы
- **order.pier** - необязательный параметр
- **order.pier.id** - ID точки отправления
- **order.pier.name** - Название точки отправления
- **order.tickets.** - Массив с билетами в заказе
- **order.tickets[].type_id** - id типа билета
- **order.tickets[].qr_code** - Контент QR кода, если есть. Необязательный параметр
- **order.tickets[].place** - Номер места, строка. Необязательный параметр
- **order.tickets[].quantity** - количество билетов в заказе

Пример ответа в случае ошибки:
```JSON
{
    "success": false,
    "message": "человекопонятное текстовое описание ошибки"
}
```

## 8. Все доступные программы

Запрос должен возвращать все доступные программы

endpoint: GET ``/programs``

Пример ответа: 
```JSON
{
    "programs": [
        {
            "id": 123,
            "name": "Ужин на теплоходе",
            "piers": [
                {
                    "id": 1,
                    "name": "Адмиралтейская набережная, 2"
                }
            ],
            "tickets": [
                {
                    "id": "123",
                    "name": "Взрослый"
                }
            ]
        }
    ]
}
```

- **programs[].tickets** - все доступные билеты для этой программы
- **programs[].piers** - все доступные причалы для этой программы
