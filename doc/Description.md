# api-json-db
## json DB с открытым кодом

«API json DB» — система управления базами данных с открытым исходным кодом которая использует JSON документы и схему базы данных. Написана на PHP. Распространяется по лицензии [MIT](https://opensource.org/licenses/MIT). Подключается через Composer как обычный пакет PHP, после подключения сама настраивается за несколько секунд. Имеет свой RESTful API интерфейс, что позволяет использовать ее с любым другим языком программирования. Фактически это менеджер json файлов с возможностью кеширования популярных запросов, проверкой валидности файлов и очередью на запись при блокировке таблиц (файлов db) на запись другими процессами. 

Главная задача «API json DB» обеспечить быструю отдачу контента (кэширование) и бесперебойную работу сайта при сбоях или недоступности API от которой она получает данные.

Ядром для «API json DB» мы выбрали прекрасную работу [Greg0/Lazer-Database](https://github.com/Greg0/Lazer-Database/). 
В связи с тем что нам необходимо добавлять новые файлы и вносить изменение в файлы мы не используем оригинальный пакет.

## Сферы применения
- Хранилище для кеша если `"cached": "true"`
- Хранилище для логов или других данных с разбивкой на файлы с лимитом записей или размером файла.
- Основная база данных с таблицами до 1 млн. записей. `"api": "false"`
- Автоматически подключение когда API недоступен. Сайт продолжает работать и пишет все `POST`, `PUT`, `DELETE` запросы в request для последующей синхронизации с API. Когда API снова станет доступный база сначала синхронизирует все данные пока не обработает и не удалит все файлы из папки request и после этого переключится на API

## Старт
Подключить пакет с помощью Composer

```json
"require": {
	"pllano/api-json-db": "*"
}
```

Сразу после установки пакета у вас есть json база данных и вы можете уже создавать таблицы и работать с db

### Прямое подключение к DB без RESTful API

Если вам не нужен API роутинг Вы можете работать с базой данных напрямую без REST API интерфейса - [Документация](https://github.com/pllano/api-json-db/blob/master/db.md) или использовать оригинальный пакет [Lazer-Database](https://github.com/Greg0/Lazer-Database/).

## RESTful API роутинг для cURL запросов

«API json DB» имеет свой RESTfull API роутинг для cURL запросов который написан на PHP с использованием Micro Framework [Slim](https://github.com/slimphp), что позволяет использовать «API json DB» с любым другим языком программирования. Для унификации обмена данными сервер-сервер и клиент-сервер используется стандарт [APIS-2018](https://github.com/pllano/APIS-2018/). `Стандарт APIS-2018 - не является общепринятым` и является нашим взглядом в будущее и рекомендацией для унификации построения легких движков интернет-магазинов нового поколения.

## RESTfull API состоит из двух файлов:
- [index.php](https://github.com/pllano/api-json-db/blob/master/index.php) - Основной файл
- [.htaccess](https://github.com/pllano/api-json-db/blob/master/.htaccess)


Если вы хотите использовать `RESTful API роутинг` выполните следующие действия:

- В файле [index.php](https://github.com/pllano/api-json-db/blob/master/index.php) указать директорию где будет находится база желательно ниже корневой директории вашего сайта. (Пример: ваш сайт находится в папке `/www/example.com/public/` разместите базу в `/www/_db_/` таким образом к ней будет доступ только у скриптов). 

- Перенесите файлы [index.php](https://github.com/pllano/api-json-db/blob/master/index.php) и [.htaccess](https://github.com/pllano/api-json-db/blob/master/.htaccess) в директорию к которой будет доступ через url (Пример: `https://example.com/_12345_/` название директории должно быть максимально сложным)

- Запустите `https://example.com/_12345_/`		
- Вы увидите следующий результат если все хорошо 
```json
{
    "status": "OK",
    "code": 200,
    "message": "db_json_api works!"
}
```

### Безопасность

При запуске база автоматически создаст файл с ключом доступа который находится в директории: `/www/_db_/core/key_db.txt`

Активация доступа по ключу устанавливается в файле [index.php](https://github.com/pllano/api-json-db/blob/master/index.php)

```php
$config['settings']['db']['access_key'] = true;
```

Если установлен флаг `true` - тогда во всех запросах к db необходимо передавать параметр `key`

`curl https://example.com/_12345_/table_name?key=key`

`curl --request POST "https://example.com/_12345_/table_name" --data "key=key"`

При запросе без ключа API будет отдавать ответ

```json
{
    "status": "403",
    "code": 403,
    "message": "Access is denied"
}
```

Вы также можете разрешить доступ к API DB только для своих IP с помощью [.htaccess](https://github.com/pllano/api-json-db/blob/master/.htaccess)

ВАЖНО !!! Не попутайте с основным `https://example.com/.htaccess` который запретит доступ ко всему сайту

В файле `https://example.com/_12345_/.htaccess` добавить

```
Order Deny,Allow
Deny from all
Allow from 192.168.1.1
```

### Пример использования с HTTP клиентом Guzzle

``` php	
$key = $config['settings']['db']['key']; // Взять key из конфигурации `https://example.com/_12345_/index.php`

$table_name = 'db';
$id = '1';

// $uri = 'https://example.com/_12345_/api.php?key='.$key;
// $uri = 'https://example.com/_12345_/'.$table_name.'?key='.$key;
$uri = 'https://example.com/_12345_/'.$table_name.'/'.$id.'?key='.$key;

$client = new \GuzzleHttp\Client();
$response = $client->request('GET', $uri);
$output = $response->getBody();

// Чистим все что не нужно, иначе json_decode не сможет конвертировать json в массив
for ($i = 0; $i <= 31; ++$i) {$output = str_replace(chr($i), "", $output);}
$output = str_replace(chr(127), "", $output);
if (0 === strpos(bin2hex($output), 'efbbbf')) {$output = substr($output, 3);}

$records = json_decode($output, true);

if (isset($records['headers']['code'])) {
if ($records['headers']['code'] == '200') {
	$count = count($records['body']['items']);
	if ($count >= 1) {
		foreach($records['body']['items'] as $item)
		{
			print_r($item['item']);
		}
	}
}
}
```

## Создание таблиц

По умолчанию при первом запуске `api-json-db` автоматически создает таблицы по стандарту `APIS-2018`, таким образом у вас есть полностью работоспособная база данных. Стуктура и взаимосвязи таблиц изменяются в файле [db.json](https://github.com/pllano/api-json-db/blob/master/_db_/core/db.json)

Вы можете создавать таблицы автоматически с использованием файла `/www/_db_/core/db.json` отредактируйте [db.json](https://github.com/pllano/api-json-db/blob/master/_db_/core/db.json) замените им `/www/_db_/core/db.json`

При запуске база проверяет файл `/www/_db_/core/db.json` и создает все таблицы которых еще не существует.

### Вы можете создавать свою конфигурацию DB для этого:
- Отредактируйте [db.json](https://github.com/pllano/api-json-db/blob/master/_db_/core/db.json)
- Удалите лишние таблицы в `/www/_db_/`

<a name="feedback"></a>
## Поддержка, обратная связь, новости

Общайтесь с нами через почту open.source@pllano.com

Если вы нашли баг в работе API json DB загляните в
[issues](https://github.com/pllano/api-json-db/issues), возможно, про него мы уже знаем и
чиним. Если нет, лучше всего сообщить о нём там. Там же вы можете оставлять свои
пожелания и предложения.

За новостями вы можете следить по
[коммитам](https://github.com/pllano/api-json-db/commits/master) в этом репозитории.
[RSS](https://github.com/pllano/api-json-db/commits/master.atom).

Лицензия API json DB
-------

The MIT License (MIT). Please see [LICENSE](LICENSE.md) for more information.
