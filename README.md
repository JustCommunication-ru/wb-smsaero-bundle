# SmsAeroBundle
Отравка sms уведомлений через сервис SmsAero. Сервис платный, необходимо иметь там учетную запись и определенную сумму денег на балансе.

Есть режим работы в качестве заглушки.

! Функционал взят из рабочего проекта без глубокой переработки структуры и зависимостей, поэтому тестно завязан не неожиданные моменты типа redis и telegram.
Требуется переработка. Когда появится такая необходимость кто-нибудь обязательно этим займется.

## Установка 
`composer require justcommunication/smsaero-bundle`

## Требования
Требует telegram-bundle

## Подключение
Прописать в /config/services.yaml:
```
parameters:
    ...    
    redis:
        parameters:
            scheme: 'tcp'
            host: '127.0.0.1'
            port: 6379
        options:
            zzz: 10
    smsaero:
        url: "http://gate.smsaero.ru/v2/"
        login: "%env(SMSAERO_LOGIN)%"
        password: "%env(SMSAERO_PASSWORD)%"
        sign: "SMS Aero" # Подпись отправителя. Необходимо зарегистрировать перед использованием
        stub: 1 # Режим заглушки. Сервис кивает хоботом, что всё отправил, логирует, но не отправляет
        log_sms: 1 # Логирование отправлений в бд (необходимо для ограничения)
        day_limit_per_ip: 20 # ограничение отправляемых смс с одного Ip за сутки работает только при log_sms: 1
```
в `.env` прописать логин/пароль от аккаунта на smsmaero.ru
```
SMSAERO_LOGIN=
SMSAERO_PASSWORD=
```

## Использование
```
autowire: SmsAeroHelper $smsAeroHelper
$smsAeroHelper->send('+79024889196', "С новым годом");
```
Обязательными являются только номер телефона и текст. Остальное уже тонкая настройка. Подробнее в описании функции:
```
/**
     * Отправка сообщений
     * @param $phone - полный номер телефона (может начинаться с +7... , 7..., 8...)
     * @param $text - текст сообщения (одна смс - 70кириллицей)
     * @param string $action - действие в проекте (для сбора статистики)
     * @param string $code - если смс для передачи кода, то указать дополнительно сюда (для статистики)
     * @param int $id_users - пользователь которому предназначается смс (для статистики)
     * @param int $try - попытка (если не первая) метод resend автоматически инкриментирует этот флаг
     * @param string $ip - ip того, кто отправил сообщение (если не указано, берется автоматически из HTTP_X_FORWARDED_FOR/REMOTE_ADDR)
     * @return bool
     * @throws \Doctrine\DBAL\Exception
     */
    public function send($phone, $text, $action='default', $code='', int $id_users=0, int $try=1, string $ip=''){
```
$smsAeroHelper
## Расширение Telegram Webhook-а
В проекте переопределить класс обработки вебкуха как описано в мануале TelegramBundle и добавить в него трейт:
```use SmsAeroWebhookTrait;```

## События SmsAero
Метод отправки сообщений генерирует одно единственное событие: превышен лимит отправки смс. Для того чтобы превратить это событие, например, в уведомление в телеграм, нужно выполнить 2 шага:

В `\config\packages\telegram.yaml` стоит добавить новое событие, хотя можно отправлять его как общий "Error"
```
    events:
        ...
        - {name: "Smsover", note: "Подписка на превышение лимита отправки смс", roles: ["Superuser"]}
```

Создать `EventSubscriber` в хост прокте. За основу можно взять `/EventSubscriber/TelegramEventSubscriber.php` лежащий в `TelegramBandle` изменив в нем событие `TelegramEvent` на `JustCommunication\SmsAeroBundle\Event\SmsAeroEvent` и название отправляемого события в ->event(). 
