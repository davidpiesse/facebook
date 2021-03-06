# Facebook Notifications Channel for Laravel 5.3 [WIP]

[![Latest Version on Packagist](https://img.shields.io/packagist/v/laravel-notification-channels/facebook.svg?style=flat-square)](https://packagist.org/packages/laravel-notification-channels/facebook)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/travis/laravel-notification-channels/facebook/master.svg?style=flat-square)](https://travis-ci.org/laravel-notification-channels/facebook)
[![StyleCI](https://styleci.io/repos/65683997/shield)](https://styleci.io/repos/65683997)
[![SensioLabsInsight](https://img.shields.io/sensiolabs/i/70841e16-34aa-496e-91c7-ba49d55841c8.svg?style=flat-square)](https://insight.sensiolabs.com/projects/70841e16-34aa-496e-91c7-ba49d55841c8)
[![Quality Score](https://img.shields.io/scrutinizer/g/laravel-notification-channels/facebook.svg?style=flat-square)](https://scrutinizer-ci.com/g/laravel-notification-channels/facebook)
[![Code Coverage](https://img.shields.io/scrutinizer/coverage/g/laravel-notification-channels/facebook/master.svg?style=flat-square)](https://scrutinizer-ci.com/g/laravel-notification-channels/facebook/?branch=master)
[![Total Downloads](https://img.shields.io/packagist/dt/laravel-notification-channels/facebook.svg?style=flat-square)](https://packagist.org/packages/laravel-notification-channels/facebook)

This package makes it easy to send notifications using the [Facebook Messenger](https://developers.facebook.com/docs/messenger-platform/product-overview) with Laravel 5.3.

## Contents

- [Installation](#installation)
	- [Setting up the Facebook service](#setting-up-the-facebook-service)
- [Usage](#usage)
	- [Available Message methods](#available-message-methods)
- [Changelog](#changelog)
- [Testing](#testing)
- [Security](#security)
- [Contributing](#contributing)
- [Credits](#credits)
- [License](#license)


## Installation

You can install the package via composer:

``` bash
composer require laravel-notification-channels/facebook
```

You must install the service provider:

```php
// config/app.php
'providers' => [
    ...
    NotificationChannels\Facebook\FacebookServiceProvider::class,
],
```

## Setting up your Facebook Bot

Follow the [Getting Started](https://developers.facebook.com/docs/messenger-platform/quickstart) guide in order to create a Facebook Messenger app, a Facebook page and a page token, which is connecting both.

Next we need to add this token to our Laravel configurations. Create a new Facebook section inside `config/services.php` and place the page token there:

```php
// config/services.php
...
'facebook' => [
    'page-token' => env('FACEBOOK_PAGE_TOKEN', 'YOUR PAGE TOKEN HERE')
],
...
```

## Usage

Let's take an invoice-paid-notification as an example.
You can now use the Facebook channel in your `via()` method, inside the InvoicePaid class. The `to($userId)` method defines the Facebook user, you want to send the notification to.

Based on the details you add (text, attachments etc.) will determine automatically the type of message to be sent. For example if you only add `text()` then it will be a basic message; using `attach()` will turn this into a attachment message. Having `buttons` or `cards` will change this to the `Button Template` and `Generic Template` respectivily

``` php
use NotificationChannels\Facebook\FacebookChannel;
use NotificationChannels\Facebook\FacebookMessage;
use NotificationChannels\Facebook\Component\Button;
use NotificationChannels\Facebook\Enums\NotificationType;

use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    public function via($notifiable)
    {
        return [FacebookChannel::class];
    }

    public function toFacebook($notifiable)
    {
        $url = url('/invoice/' . $this->invoice->id);

        return FacebookMessage::create()
            ->to($this->user->fb_messenger_user_id) // Optional
            ->text('One of your invoices has been paid!')
            ->notificationType(NotificationType::REGULAR) // Optional
            ->buttons([
                Button::create('View Invoice', $url)->isTypeWebUrl(),
                Button::create('Call Us for Support!', '+1(212)555-2368')->isTypePhoneNumber(),
                Button::create('Start Chatting', ['invoice_id' => $this->invoice->id])->isTypePostback() // Custom payload sent back to your server
            ]); // Buttons are optional as well.
    }
}
```

The notification will be sent from your Facebook page, whose page token you have configured earlier. Here's a screenshot preview of the notification inside the chat window.

![Laravel Facebook Notification Example](https://cloud.githubusercontent.com/assets/1915268/17666125/58d6b66c-631c-11e6-9380-0400832b2e48.png)

#### Message Examples
##### Basic Text Message
Send a basic text message to a user
```php
return FacebookMessage::create('You have just paid your monthly fee! Thanks')
->to($this->user->fb_messenger_id);
```
##### Attachment Message
Send a file attachment to a user (Example is sending a pdf invoice)
```php
return FacebookMessage::create()
->attach(AttachmentType::FILE, url('invoices/'.$this->invoice->id))
->to($this->user->fb_messenger_id);
```

##### Generic (Card Carousel) Message
Send a set of cards / items to a user displayed in a carousel (Example is sending a set of links). Note you can also add up to three buttons per card
```php
return FacebookMessage::create()
->cards([
    Card::create('Card No.1 Title')
    ->subtitle('An item description')
    ->url('items/'.$this->item[0]->id)
    ->image('items/'.$this->item[0]->id.'/image'),
    Card::create('Card No.2 Title')
    ->subtitle('An item description')
    ->url('items/'.$this->item[1]->id)
    ->image('items/'.$this->item[1]->id.'/image')
    //could add buttons using ->buttons($array of Button)
])
->to($this->user->fb_messenger_id);
```

### Routing a message

You can either send the notification by providing with the page-scoped user id (PSID) of the recipient to the `to($userId)` method like shown in the above example or add a `routeNotificationForFacebook()` method in your notifiable model:

``` php
...
/**
 * Route notifications for the Facebook channel.
 *
 * @return int
 */
public function routeNotificationForFacebook()
{
    return $this->fb_messenger_user_id;
}
...
```

### Available Message methods

- `to($userIdOrPhoneNumber)`: (string) Recipient's page-scoped User ID or Phone number of with the format `+1(212)555-2368`. **NOTE:** Sending a message to phone numbers requires the `pages_messaging_phone_number` permission. Refer [docs](https://developers.facebook.com/docs/messenger-platform/send-api-reference#phone_number) for more information.
- `text('')`: (string) Notification message.
- `attach($attachment_type, $url)`: (AttachmentType, string) An attachment type (IMAGE, AUDIO, VIDEO, FILE) and the url of this attachment
- `buttons($buttons = [])`: (array) An array of "Call to Action" buttons (Created using `NotificationChannels\Facebook\Components\Button::create()`). You can add up to 3 buttons of one of the following types: `web_url`, `postback` or `phone_number`. See Button methods below for more details.
- `cards($cards = [])`: (array) An array of item cards to be displayed in a carousel (Created using `NotificationChannels\Facebook\Components\Card::create()`). You can add up to 10 cards. See Card methods below for more details.
- `notificationType('')`: (string) Push Notification type: `REGULAR` will emit a sound/vibration and a phone notification; `SILENT_PUSH` will just emit a phone notification, `NO_PUSH` will not emit either. You can make use of `NotificationType::REGULAR`, `NotificationType::SILENT_PUSH` and `NotificationType::NO_PUSH` to make it easier to work with the type. This is an optional method, defaults to `REGULAR` type.

### Available Button methods

- `title('')`: (string) Button Title.
- `data('')`: (string) Button Data - It can be a web url, postback data or a formated phone number.
- `type('')`: (string) Button Type - `web_url`, `postback` or `phone_number`. Use `ButtonType` enumerator for guaranteeing valid values
- `isTypeWebUrl()`: Helper method to create a `web_url` type button.
- `isTypePhoneNumber()`: Helper method to create a `phone_number` type button.
- `isTypePostback()`: Helper method to create a `postback` type button.

### Available Card methods

- `title('')`: (string) Card Title.
- `subtitle('')`: (string) Card Subtitle.
- `url('')`: (string) Card Item Url.
- `image('')`: (string) Card Image Url. Image ratio should be 1.91:1
- `buttons($buttons = [])`: (array) An array of "Call to Action" buttons (Created using `NotificationChannels\Facebook\Components\Button::create()`). You can add up to 3 buttons of one of the following types: `web_url`, `postback` or `phone_number`. See Button methods above for more details.
-
## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Credits

- [Syed Irfaq R.](https://github.com/irazasyed)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
