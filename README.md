# Содержание

* [Общие принципы отправки](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%9E%D0%B1%D1%89%D0%B8%D0%B5-%D0%BF%D1%80%D0%B8%D0%BD%D1%86%D0%B8%D0%BF%D1%8B-%D0%BE%D1%82%D0%BF%D1%80%D0%B0%D0%B2%D0%BA%D0%B8)
    * [Пример передачи XML документа на phр](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%9F%D1%80%D0%B8%D0%BC%D0%B5%D1%80-%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%87%D0%B8-xml-%D0%B4%D0%BE%D0%BA%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D0%B0-%D0%BD%D0%B0-php)
    * [Пример многопоточной передачи XML документа на php](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%9F%D1%80%D0%B8%D0%BC%D0%B5%D1%80-%D0%BC%D0%BD%D0%BE%D0%B3%D0%BE%D0%BF%D0%BE%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B9-%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%87%D0%B8-xml-%D0%B4%D0%BE%D0%BA%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D0%B0-%D0%BD%D0%B0-php)
* [Отправка SMS, Flash SMS, WAP-Push](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%9E%D1%82%D0%BF%D1%80%D0%B0%D0%B2%D0%BA%D0%B0-sms-flash-sms-wap-push)
* [Запрос статуса SMS сообщения (первый способ)](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D1%81%D1%82%D0%B0%D1%82%D1%83%D1%81%D0%B0-sms-%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B5%D0%BD%D0%B8%D1%8F-%D0%BF%D0%B5%D1%80%D0%B2%D1%8B%D0%B9-%D1%81%D0%BF%D0%BE%D1%81%D0%BE%D0%B1)
* [Получение статуса SMS сообщения (второй способ)](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%9F%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D1%81%D1%82%D0%B0%D1%82%D1%83%D1%81%D0%B0-sms-%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B5%D0%BD%D0%B8%D1%8F-%D0%B2%D1%82%D0%BE%D1%80%D0%BE%D0%B9-%D1%81%D0%BF%D0%BE%D1%81%D0%BE%D0%B1)
* [Запрос проверки баланса](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B8-%D0%B1%D0%B0%D0%BB%D0%B0%D0%BD%D1%81%D0%B0)
* [Запрос на получение списка отправителей](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D1%8F-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%B0-%D0%BE%D1%82%D0%BF%D1%80%D0%B0%D0%B2%D0%B8%D1%82%D0%B5%D0%BB%D0%B5%D0%B9)
* [Запрос входящих SMS](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%B2%D1%85%D0%BE%D0%B4%D1%8F%D1%89%D0%B8%D1%85-sms)
* [Запрос на получение информации по номеру телефона](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%86%D0%B8%D0%B8-%D0%BF%D0%BE-%D0%BD%D0%BE%D0%BC%D0%B5%D1%80%D1%83-%D1%82%D0%B5%D0%BB%D0%B5%D1%84%D0%BE%D0%BD%D0%B0)
* [Запрос на получение списка баз]()
* [Запрос на изменение параметров/добавление/удаление баз]()
* [Запрос на получение списка абонентов базы](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%B0-%D0%B0%D0%B1%D0%BE%D0%BD%D0%B5%D0%BD%D1%82%D0%BE%D0%B2-%D0%B1%D0%B0%D0%B7%D1%8B)
* [Запрос на добавление/редактирование/удаление абонентов базы](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%B4%D0%BE%D0%B1%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%D1%83%D0%B4%D0%B0%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B1%D0%BE%D0%BD%D0%B5%D0%BD%D1%82%D0%BE%D0%B2-%D0%B1%D0%B0%D0%B7%D1%8B)
* [Запрос на получение списка номеров из СТОП-листа](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%B0-%D0%BD%D0%BE%D0%BC%D0%B5%D1%80%D0%BE%D0%B2-%D0%B8%D0%B7-%D0%A1%D0%A2%D0%9E%D0%9F-%D0%BB%D0%B8%D1%81%D1%82%D0%B0)
* [Запрос на добавление/удаление абонентов в СТОП-лист](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%B0-%D0%B7%D0%B0%D0%BF%D0%BB%D0%B0%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-sms)
* [Запрос на получение списка запланированных SMS](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D1%81%D0%BF%D0%B8%D1%81%D0%BA%D0%B0-%D0%B7%D0%B0%D0%BF%D0%BB%D0%B0%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-sms)
* [Запрос на удаление запланированной SMS](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D1%83%D0%B4%D0%B0%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B7%D0%B0%D0%BF%D0%BB%D0%B0%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D0%BE%D0%B9-sms)
* [Запрос на получение времени изменения чего-либо](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BD%D0%B0-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%B8-%D0%B8%D0%B7%D0%BC%D0%B5%D0%BD%D0%B5%D0%BD%D0%B8%D1%8F-%D1%87%D0%B5%D0%B3%D0%BE-%D0%BB%D0%B8%D0%B1%D0%BE)
* [Запрос проверки времени](https://github.com/smsgate/xmlGateDoc/blob/master/README.md#%D0%97%D0%B0%D0%BF%D1%80%D0%BE%D1%81-%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B8-%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%B8)

# Общие принципы отправки
На  определенный   адрес   сервера   отправляются   XML   документы   (описание   XML документов, их назначение и адреса сервера приведены ниже). При этом используется POST метод.

Заголовки отправляемых данных должны содержать:
```
Content-type: text/xml; charset=utf-8
```
Кодировка XML документов UTF-8
При этом передаваемый XML документ не должен содержать переводов строки. Переводы строк в самих данных должны быть заменены на “\n”.
## Пример передачи XML документа на php
```php
$src = '<?xml version="1.0" encoding="utf-8"?>
<request>
<security>
<login value="логин" />
<password value="пароль" />
</security>
</request>'; 
// XML-документ
$href = 'http://server/script.php'; // адрес сервера
$ch = curl_init();
curl_setopt ($ch, CURLOPT_HTTPHEADER, array ('Content-type: text/xml; charset=utf-8')); 
curl_setopt ($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
curl_setopt ($ch, CURLOPT_CRLF, true); 
curl_setopt ($ch, CURLOPT_POST, true); 
curl_setopt ($ch, CURLOPT_POSTFIELDS, $src); 
curl_setopt ($ch, CURLOPT_URL, $href);
$result = curl_exec($ch); 
curl_close($ch);
echo $result;
```
## Пример многопоточной передачи XML документа на php
Если нет возможности собирать запросы в один xml, можно отправлять много xml за раз через curl_multi http://github.com/LionsAd/rolling-curl . При  передачи 5 смс (1 смс 1 xml) этот метод быстрее до 3,5 раз. Не рекомендуется ставить больше 5 потоков, возможны потеря запросов с данными. Порядок запросов при таком методе не соблюдается.
```php
<?php
include_once './rolling-curl-read-only/RollingCurl.php';

$arr_xml = array(
    '<?xml version="1.0" encoding="utf-8"?><request><security><login value="логин" /><password value="пароль"/></security></request>',
    '<?xml version="1.0" encoding="utf-8"?><request><security><login value="логин" /><password value="пароль" /></security></request>',
);

$opt = array(
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_CRLF => true,
    CURLOPT_POST => true,
    CURLOPT_SSL_VERIFYHOST => false,
    CURLOPT_HTTPHEADER =>  array('Content-type: text/xml; charset=utf-8'),
);

$rc = new RollingCurl("request_callback");
$rc->window_size = 5; //колличество потоков
foreach ($arr_xml as $data) {
    $request = new RollingCurlRequest("http://server/script.php", "POST", $data, null, $opt);
    $rc->add($request);
}
$rc->execute();

function request_callback($response, $info, $request) {
    print_r($response); //ответ сервера
}
```
## Отправка SMS, Flash SMS, WAP-Push
**Адрес сервера:**
```
https://имя_хоста/xml/
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?> 
<request>
<message type="flashsms или sms или wappush или vcard">
    <sender>Отправитель 1</sender>
    <text>Текст сообщения 1</text>
    <url>Адрес для WAP Push или vCard</url>
    <name>Имядля vCard</name>
    <phone cell="79033256699" work="79033256699" fax="79033256699"/>
    <email>E-mail vCard</email>
    <position>Должность vCard</position>
    <organization>Организация vCard</organization>
    <address post_office_box="абонентскийящик" street="Улица" city="город" region="Область" postal_code="Индекс" country="Страна" />
    <additional>Дополнительнаяинформация vCard</additional>
    <abonent phone="79033256699" number_sms="1" client_id_sms="101" time_send="2001-12-31 12:34" validity_period="2001-12-31 15:34" />
    <abonent phone="79033256699" number_sms="2" client_id_sms="102" time_send="2001-12-31 12:35" />
    <abonent phone="79033256699" number_sms="10" client_id_sms="110" time_send="" />
</message>
<message>
    <sender>Отправитель 2</sender>
    <text>Текстсообщения 2</text>
    <abonent phone="79033256699" number_sms="11" client_id_sms="111" />
    <abonent phone="79033256699" number_sms="12" client_id_sms="112" />
    <abonent phone="79033256699" number_sms="20" client_id_sms="120" />
</message>

<security>
    <login value="логин" />
    <password value="пароль" /> 
</security>
</request>
```
Где
* **type** - тип отправляемого SMS сообщения:
    * **flashsms**  – flash SMS
    * **sms**  – обычная SMS
    * **wappush**  – WAP-Push
    * **vcard**  – визитная карточка (vCard) 
* **sender** – отправитель SMS. Именно это значение будет выводиться на телефоне абонента в поле от кого SMS. 
* **phone** – номер абонента, которому адресована SMS.
* **loginvalue** - ваш логин в системе
* **passwordvalue** - ваш пароль в системе
* **number_sms**  - номер сообщения в пределах отправляемого XML документа
* **client_id_sms** - число. Необязательный параметр, позволяет избежать повторной отправки. Если раннее с этого аккаунта уже было отправлено SMS с таким номером, то повторная отправка не производится, а возвращается номер ранее отправленного SMS. 
* **time_send** – дата и время отправки в формате: YYYY-MM-DD hh:mm где, YYYY-год, MM-месяц, DD-день, hh-часы, mm-минуты. Если не задано, то SMS отправляется сразу же. 
* **validity_period** – дата и время, после которых не будут делаться попытки доставить SMS в формате:  YYYY-MM-DD hh:mm , где YYYY-год, MM-месяц, DD-день, hh-часы, mm-минуты. Если не задано, то SMS имеет максимальный срок жизни. 

Далее поля выбираются в зависимости от типа, отправляемого SMS (type):
* **text** - текст обычного SMSили описание WAPссылки
* **url** - ссылка для WAP Push или vCard
* **name** - имя для vCard
* **cell** - номер телефона для vCard
* **work** - номер рабочего телефона для vCard
* **fax** - номер факса для vCard
* **email** - e-mail для vCard
* **position** - должность контакта для vCard
* **organization** – организация для vCard
* **post_office_box** - абонентский ящик для vCard
* **street** - улица для vCard
* **city** - город для vCard
* **region** - область для vCard
* **postal_code** - индекс для vCard
* **country** - страна для vCard
* **additional** - дополнительная информация для vCard

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
<error>текст ошибки</error>
</response> 
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа
2. Ваш аккаунт заблокирован
3. Неправильный логин или пароль
4. POST данные отсутствуют

### В случае получения правильного XML-документа
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<information number_sms="1" id_sms="ID SMS в системе для проверки статуса" parts="2">Статус/сообщение об ошибке</information>
<information number_sms="2" id_sms="ID SMS в системе для проверки статуса" parts="2">Статус/сообщение об ошибке</information>
<information number_sms="3" id_sms="ID SMS в системе для проверки статуса" parts="2">Статус/сообщение об ошибке</information>
</response>
```
Где:
* **number_sms** - номер сообщения указанный при отправке XML документа.
* **id_sms** - номер SMS сообщения. Используется для проверки статуса SMS. Если в процессе отправки SMS произошла ошибка, то id_sms не передается. 
* **parts** - количество частей SMS.
* **information** -   статус   сообщения   («send»),   если   SMS   была   отправлена.   Или сообщение об ошибке, если в процессе отправки SMS произошла ошибка: 
    1. У нас закончились SMS. Для разрешения проблемы свяжитесь с менеджером.
    2. Закончились SMS.
    3. Аккаунт заблокирован.
    4. Укажите номер телефона.
    5. Номер телефона присутствует в стоп-листе.
    6. Данное направление закрыто для вас.
    7. Данное направление закрыто.
    8. Текст SMS отклонен модератором.
    9. Нет отправителя.
    10. Отправитель не должен превышать 15 символов для цифровых номеров и 11 символов для буквенно-числовых.
    11. Номер телефона должен быть меньше 15 символов.
    12. Нет текста сообщения.
    13. Нет ссылки.
    14. Укажите название контакта и, хотя бы один параметр для визитной карточки.
    15. Такого отправителя Нет.
    16. Отправитель не прошел модерацию.

# Запрос статуса SMS сообщения (первый способ)

**Адрес сервера:**
```
https://имя_хоста/xml/state.php
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?> 
<request>
<security>
    <login value="логин" /> 
    <password value="пароль" />
</security>
<get_state>
    <id_sms>IDSMS в системе для проверки статуса</id_sms> 
    <id_sms>IDSMS в системе для проверки статуса</id_sms>
    <id_sms>IDSMS в системе для проверки статуса</id_sms>
    <id_sms>IDSMS в системе для проверки статуса</id_sms>
</get_state>
</request>
```
Где
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **id_sms** - номер SMS сообщения, полученный в ответном XML-документа в процессе отправки SMS сообщения. 

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа
2. Неправильный логин или 
3. POST данные отсутствуют 

### В случае получения правильного XML-документа:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<state id_sms="IDSMS в системе для проверки статуса" time="2011-01-01 12:57:46">Статус</state>
<state id_sms="IDSMS в системе для проверки статуса" time="2011-01-01 12:57:46">Статус</state>
<state id_sms="IDSMS в системе для проверки статуса" time="2011-01-01 12:57:46">Статус</state>
</response>
```
Где
* **id_sms** - номер   SMS   сообщения,   полученный   в   ответном   XML-документа   в процессе отправки SMS сообщения.
* **time** - время изменения статуса.
* **state** - статус сообщения:
    1. «send» - статус сообщения не получен. В этом случае передается пустой time (time="").
    2. «not_deliver»  - сообщение   не   было   доставлено.   Конечный   статус   (не меняется со временем).
    3. «expired» - абонент находился не в сети в те моменты, когда делалась попытка доставки. Конечный Статус (не меняется со временем.
    4. «deliver»   -   сообщение   доставлено.   Конечный   статус (не   меняется   со временем)
    5. «partly_deliver» - сообщение было отправлено, но статус так и не был получен. Конечный статус (не меняется со временем). В этом случае для разъяснения причин отсутствия статуса необходимо связаться со службой тех. поддержки. 

# Получение статуса SMS сообщения (второй способ)
При использовании данного способа необходимо сообщить менеджеру адрес вашего сервера, который будет принимать статусы SMS.
XML-документ будет отправлен POST методом. 

К примеру, в php XML-документ будет доступен через переменную
```php
$GLOBALS['HTTP_RAW_POST_DATA']
```
Система отправляет XML-документ серверу клиента следующего содержания:
```xml
<?xml version="1.0" encoding="utf-8"?>
<request>
    <state id_sms="ID SMS в системе для проверки статуса" time="2011-01-01 12:57:46">Статус</state>
    <state id_sms="ID SMS в системе для проверки статуса" time="2011-01-01 12:57:46">Статус</state>
</request>
```
Где:
* **id_sms** - номер   SMS   сообщения,   полученный   в   ответном   XML-документа   в процессе отправки SMS сообщения.
* **state** - статус сообщения:
    1. «send» - статус сообщения не получен. В этом случае передается пустой time (time=""). 
    2. «not_deliver» - сообщение не было доставлено. Конечный статус (не меняется со временем). 
    3. «expired»  - абонент находился не  в сети в  те моменты,  когда делалась  попытка доставки. Конечный Статус (не меняется со временем). 
    4.  «deliver» - сообщение доставлено. Конечный статус (не меняется со временем). 
    5. «partly_deliver» - сообщение было отправлено, но статус так и не был получен. Конечный статус (не меняется со временем). В этом случае для разъяснения причин отсутствия статуса необходимо связаться со службой тех. поддержки. 

В ответ сервер клиента должен вернуть XML-документ следующего содержания:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <id_sms>3234</id_sms>
    <id_sms>3234</id_sms>
</response>
```
Где:
* **id_sms** - номер   SMS   сообщения,   полученный   в   ответном   XML-документа   в процессе отправки SMS сообщения. 
* **time** - время изменения статуса.

Если сервер клиента не передаст id_sms, то статус будет считаться не полученным  клиентом. При этому будет сделано 5 попыток доставить статус.
# Запрос проверки баланса
**Адрес сервера:**
```
https://имя_хоста/xml/balance.php
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
</request>
```
Где
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе

В ответ может быть выдан один из следующих XML-документов: 
1. В случае возникновения ошибки в отправляемом XML-документе: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
2. В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <money currency="RUR">150</money>
    <sms area="Россия">111</sms>
    <sms area="Украина">111</sms>
</response>
```
Где:
* **money** - остаток средств.
* **area** - направление в котором может быть отправлено данное количество SMS.
* **sms** - количество доступных SMS сообщений для данного направления.

При этом количество SMS не может быть суммировано по разным направлениям. При отправке смс в одном направлении уменьшается количество доступных SMS сообщений во всех других направлениях в соответствии с их стоимостью.

**Пример**

У вас на балансе 10 y.e.

Стоимость SMS сообщения отправленного в Россию составляет 1 y.e.

Стоимость SMS сообщения отправленного в Украину составляет 2 y.e.

При этом вам вернется XML документ следующего содержания.
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<money>10</money>
<sms area="Россия">10</sms>
<sms area="Украина">5</sms>
</response>
```
Если вы отправите две смс в Россию, то XML-документ изменится следующим образом:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <money>8</money>
    <sms area="Россия">8</sms>
    <sms area="Украина">4</sms>
</response>
```
# Запрос на получения списка отправителей
**Адрес сервера:**
```
https://www.имя_хоста/xml/originator.php
```
***XML-документ:***
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
error - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <any_originator>FALSE</any_originatоr>
    <list_originator>
        <originator state="rejected">Отправитель</originator>
    </list_originator>
</response>
```
Где
* **any_originator** - TRUE/FALSE - может ли клиент отправлять от любого отправителя или только от заранее одобренных. Если TRUE, то клиент может использовать любого отправителя. При этом список отправителей не возвращается.  FALSE – можно использовать только отправителей со статусом «completed». 
* **state** - статус отправителя:
    1. order – оформляется 
    2. completed - готов к использованию 
    3. rejected – отклонен 

# Запрос входящих SMS
**Адрес сервера:** 
```
https://имя_хоста/xml/incoming.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
    <security>
        <login value="логин" />
        <password value="пароль" />
    </security>
<time start="2012-01-31 12:23:00" end="2012-02-31 12:23:00" />
</request>
```
Где
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **time start** - время (не включительно), с которого запрашиваются входящие SMS.
* **time end** - время (не включительно), по которое запрашиваются входящие SMS. Не обязательный параметр. Если не задан, то будут возвращены все смс.

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 

### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?> 
<response>
<sms id_sms="1234" date_receive="2012-01-31 12:55:55" originator="79612242243" prefix="IGRA"phone="79611111111">ТекстСМС.</sms>
<sms id_sms="1234" date_receive="2012-01-31 12:55:55" originator="79612242243" prefix="IGRA"phone="79611111111">ТекстСМС.</sms>
</response>
```
Где
* **id_sms** - уникальный номер, состоящий только из цифр.
* **date_receive** - дата и время получения SMS.
* **originator** - номер телефона абонента, отправившего SMS.
* **prefix** - префикс. Начальная часть текста SMS, по которой было определено, что эта SMS принадлежит именно этому клиенту. (Используется если один и тот же номер используется разными клиентами.) 
* **phone** - номер телефона, на который бала отправлена SMS.
* **sms** - текст смс

# Запрос на получение информации по номеру телефона
**Адрес сервера:**
```
https://имя_хоста/xml/def.php
```
**XML-документ:**
```
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<phones>
    <phone>79612242243</phone>
    <phone>79612242244</phone>
</phones>
</request>
```
Где: 
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **phone** - Номер телефона

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
<error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <phone operator="Вымпелтелеком" region="Новосибирскаяобласть" time_zone="3">79612242243</phone>
    <phone operator="Вымпелтелеком" region="Калининград" time_zone="-1">79612242244</phone>
</response>
```
Где:
* **operator** - Оператор.
* **region** - Регион.
* **time_zone** - Смещение времени в часах относительно времени в Москве.
* **phone**  - номер телефона.

# Запрос на получение списка баз
**Адрес сервера:**
```
https://имя_хоста/xml/list_bases.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
    <security>
        <login value="логин" />
        <password value="пароль" />
    </security>
</request>
```
Где:
* **login value** - ваш логин в системе
*  **password value** - ваш пароль в системе

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response> 
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 

### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<base id_base="1234" name_base="Базаглавногоофиса" time_birth="12:48" local_time_birth="yes" day_before="1" originator_birth="fitnes" on_birth="yes">Поздравляем!</base>
<base id_base="1235" name_base="БазаМосковскогоофиса" time_birth="12:48" local_time_birth="yes" day_before="1" originator_birth="fitnes" on_birth="yes">Поздравляем!</base>
</response>
```
Где:
* **id_base** - уникальный номер базы в системе.
* **name_base** - название базы.
* **time_birth** - время поздравления.
* **local_time_birth** - ччитать время поздравления относительно местного времени абонента(yes) или относительно времени системы (no). 
* **day_before** - за сколько дней до дня рождения поздравлять.
* **originator_birth** - отправитель поздравления.
* **on_birth** - включены ли поздравления yes – включены, no - выключены.
* **base** - текст поздравления.

# Запрос на изменение параметров/добавление/удаление баз
**Адрес сервера:**
```
https://имя_хоста/xml/bases.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" /> 
</security>
<bases>
<base id_base="1234" name_base="Базаглавногоофиса" time_birth="12:48" local_time_birth="yes" day_before="1" originator_birth="fitnes" on_birth="yes">Поздравляем!</base>
<base number_base="1" name_base="БазаМосковскогоофиса" time_birth="12:48" local_time_birth="yes" day_before="1" originator_birth="fitnes" on_birth="yes">Поздравляем!</base>
</bases>
<delete_bases>
    <base id_base="1235" />
    <base id_base="1236" />
</delete_bases>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **id_base** - уникальный номер базы в системе. Если не указан произойдет добавление базы. При этом нужно указать параметр number_base. 
* **number_base** - номер базы в XML запросе. Указывается только при создании новой базы. Используется для сопоставления ID добавленных баз (если их было не сколько в  запросе). 
* **name_base** - название базы.
* **time_birth** - время поздравления.
* **local_time_birth** - считать время поздравления относительно местного времени абонента (yes) или относительно времени системы (no). 
* **originator_birth** - отправитель поздравления.
* **on_birth** - включены ли поздравления yes – включены, no - выключены.
* **base** - текст поздравления.

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 

### В случае получения правильного XML-документа:
```
<?xml version="1.0" encoding="utf-8" ?>
<response>
<base id_base=”1234”>edit</base>
<base number_base=”1” id_base=”1235”>insert</ base>
<base number_base=”2” id_base=”1236”>edit</ base>
<base id_base=”1235”>delete</ base>
<base id_base=”1235”>not_found</ base>
</response>
```
# Запрос на получение списка абонентов базы
**Адрес сервера:** 
```
https://имя_хоста/xml/list_phones.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<base id_base="1234" page=”1” last_update=”2011-03-25 08:39:48”/>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **baseid_base** - номер базы в системе.
* **basepage** -  номер   страницы.   Весь   список   номеров   базы   делится   на страницы. Запросить   целиком   базу   нельзя.   Можно   лишь   запросить отдельную страницу. Нумерация начинается с единички. 
* **last_update** - минимальная   дата   и   время   регистрации   (или   последнего изменения) данных абонента, которых Вам нужно запросить. 

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
4. Базы с таким номером не существует 

### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<phones page="1" num_pages="100">
<phone phone="79612242243" region="Новосибирск " operator="Вымпелком" name="Константин" surname="Ермолаев" patronymic="Александрович" date_birth="1984-08-21" male="m" addition_1="Первоедополнительноеполе" addition_2="второе" last_update=” 2011-03-25 08:39:48” />
<phone phone="79612242244" region="Новосибирск» operator="Вымпелком" name="Константин" surname="Ермолаев" patronymic="Александрович" date_birth="1984-08-21" male="m" addition_1="Первоедополнительноеполе" addition_2="второе" last_update=” 2011-03-25 08:39:48” />
</phones>
</response>
```
Где:
* **page** - номер страницы
* **num_pages** - всего страниц
* **phone** - номер телефона абонента
* **region** - регион
* **operator** - оператор
* **name** - имя абонента
* **surname** - фамилия абонента
* **patronymic** - отчество абонента
* **date_birth** - дата рождения
* **male** - пол. «m» - мужской, «f»- женский
* **addition_1** – первое дополнительное поле.
* **addition_2** – второе дополнительное поле.
* **last_update** - дата   и   время   регистрации   (или   последнего   изменения)   данных абонента. 

# Запрос на добавление/редактирование/удаление абонентов базы
**Адрес сервера:**
```
https://имя_хоста/xml/phones.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?> 
<request>
<security>
    <login value="логин" /> 
    <password value="пароль" />
</security>
<base id_base="1234">
<phone phone="79612242243" region="Новосибирск " operator="Вымпелком" name="Константин" surname="Ермолаев" patronymic="Александрович" date_birth="1984-08-21" male="мужской" addition_1="Первоедополнительноеполе" addition_2="второе" 
number_phone="1"/>
<phone phone="79612242244" region="Новосибирск " operator="Вымпелком" name="Константин" surname="Ермолаев" patronymic="Александрович" date_birth="1984-08-21" male="мужской" addition_1="Первоедополнительноеполе" addition_2="второе" 
number_phone="2" />
<phone phone="79612242243" action="delete" number_phone="5"/>
<phone phone="79612242244" action="delete" number_phone="6"/>
</base>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **base id_base** - номер базы в системе.
* **phone** - номер телефона абонента. Если абонент с таким номером уже существует, то он будет отредактирован.
* **region** - регион. Необязательное поле. Если не задано определяется автоматически.
* **operator** - оператор.   Необязательное   поле.   Если   не   задано   определяется автоматически.
* **name** - имя абонента. Необязательное поле.
* **surname** - фамилия абонента. Необязательное поле.
* **patronymic** - отчество абонента. Необязательное поле.
* **date_birth** - дата рождения. Необязательное поле.
* **male** - пол. «мужской» или «женский». Необязательное поле.
* **addition_1** - первое дополнительное поле. Необязательное поле.
* **addition_2** - второе дополнительное поле. Необязательное поле.

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?> 
<response> 
    <error>текстошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
4. Базы с таким номером не существует 

### В случае получения правильного XML-документа:
```
<?xml version="1.0" encoding="utf-8" ?>
<response>
<baseid_base="1234">
<phone phone="79612242243" number_phone="1"/>insert</phone>
<phone phone="79612242244" number_phone="2" />edit</phone>
<phone phone="79612242243" number_phone="5"/>delete</phone> 
<phone phone="79612242244" number_phone="6" />not_found</phone>
</response>
```
# Запрос на получение списка номеров из СТОП-листа
**Адрес сервера:**
```
https://имя_хоста/xml/list_stop.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" /> <password value="пароль" />
</security>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
<error>текстошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 

### В случае получения правильного XML-документа:
```
<?xml version="1.0" encoding="utf-8" ?>
<response>
<phone>79612242243</phone>
<phone>79612242244</phone>
</response>
```

Где:
* **phone** - номер телефона из СТОП-листа.
# Запрос на добавление/удаление абонентов в СТОП-лист
**Адрес сервера:** 
```
https://имя_хоста/xml/stop.php
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<add_stop>
    <phone phone="79612242243” />
    <phone phone="79612242244" />
</add_stop>
<delete_stop>
    <phone phone="79612242243” />
    <phone phone="79612242244" />
</delete_stop>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **add_stop phone** - номер телефона абонента, которого нужно добавить в СТОП-лист.
* **delete_stop phone** - номер телефона абонента, которого нужно удалить из СТОП-листа. 

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <phone phone="79612242243">delete</phone>
    <phone phone="79612242244">add</phone>
    <phone phone="79612242245">not_found</phone>
</response>
```
# Запрос на получение списка запланированных SMS
**Адрес сервера:** 
```
https://имя_хоста/xml/list_scheduled.php
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<scheduled page=”1”/>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **scheduled page** - номер страницы. Весь  список запланированных SMS делится   на страницы.   Запросить   список   целиком   нельзя.   Можно   лишь запросить отдельную страницу. Нумерация начинается с единички. 

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<phones page="1" num_pages="100">
<scheduled id_sms="1234" time_put_turn="2011-11-14 12:42:40" originator="kosty"  phone="79612242243" type_sms="sms" text_sms="ТекстSMS" count_sms="2" name_delivery="Названиерасылки" time_send="2011-11-14 21:00" validity_period="2011-11-17 21:00:00" />
<scheduled id_sms="1235" time_put_turn="2011-11-14 12:42:40" originator="kosty" phone="79612242244" type_sms="sms" text_sms="ТекстSMS" count_sms="2" name_delivery="Названиерасылки" time_send="2011-11-14 21:00" validity_period="2011-11-17 21:00:00" />
</phones>
</response>
```
Где:
* **page** - номер страницы
* **num_pages** - всего страниц
* **id_sms** - номерSMS. Используется для удаления запланированной SMS.
* **time_put_turn** - время добавления в планировщик.
* **operator** – отправитель SMS. Именно это значение будет выводиться на телефоне абонента в поле от кого SMS. 
* **phone** - номер абонента, которому адресована SMS.
* **type** – тип отправляемого SMS сообщения:
    * sms – обычная SMS
    * flashsms – flash SMS
    * wappush – WAP-Push
    * vcard – визитная карточка (vCard) 
* **text_sms** - Текст SMS-сообщения.
* **count_sms** –Количество частей SMS-сообщения.
* **name_delivery**  - Название рассылки.
* **time_send** - дата и время отправки в формате: YYYY-MM-DDHH:MM где, YYYY-год, MM-месяц, DD-день, HH-часы, MM-минуты. 
* **validity_period** - дата и время, после которых не будут делаться попытки доставить SMS в формате: YYYY-MM-DDHH:MM:SS где, YYYY-год, MM-месяц, DD-день, HH-часы, MM-минуты, SS-секунды. 

# Запрос на удаление запланированной SMS
Адрес сервера: 
```
https://имя_хоста/xml/scheduled.php
```
**XML-документ:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<delete_schedule>
    <schedule id_sms="1234” />
    <schedule id_sms="1235” />
</delete_schedule>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **delete_schedule id_sms** - номер запланированной SMS, которую нужно удалить. Можно получить при запросе списка запланированных SMS. 

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текстошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <scheduled id_sms="1234">delete</scheduled> 
    <scheduled id_sms="1235">not_found</scheduled>
</response>
```
# Запрос на получение времени изменения чего либо
Адрес сервера: 
```
https://www.имя_хоста/xml/check_change.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
<check obgect="base" id="1"/>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе
* **obgect - base** - базы данных, stop- лист

В ответ может быть выдан один из следующих XML-документов: 
### В случае возникновения ошибки в отправляемом XML-документе:
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>текст ошибки</error>
</response>
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
4. Базы с таким номером не существует 

### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
    <obgect time_update="2012-01-01 12:12:12" />
</response>
```
Где
* **time_update** - время последнего изменения объекта

# Запрос проверки времени
**Адрес сервера:**
```
https://имя_хоста/xml/time.php
```
XML-документ:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<request>
<security>
    <login value="логин" />
    <password value="пароль" />
</security>
</request>
```
Где:
* **login value** - ваш логин в системе
* **password value** - ваш пароль в системе

В ответ может быть выдан один из следующих XML-документов: 

### В случае возникновения ошибки в отправляемом XML-документе: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response> 
<error>текстошибки</error>
</response> 
```
**error** - текст ошибки может принимать следующие значения:
1. Неправильный формат XML документа 
2. Неправильный логин или пароль 
3. POST данные отсутствуют 
### В случае получения правильного XML-документа:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<response>
<time>15:34:05</time>
</response>
```
Где:
* **time** - локальное время пользователя.

При этом время рассчитывается как время на сервере минус разница часовых поясов 
пользователя и сервера.

### Пример
Сервер находится в Москве и серверное время у него московское, в момент запроса оно 
составляло: 15:34:27. А пользователь числился в Новосибирске и разница часовых поясов (между сервером и 
пользователем) у него равнялась +3. При этом вам вернется XML документ следующего содержания: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<response>
    <time>2012-12-17 18:34:27</time>
</respo
```