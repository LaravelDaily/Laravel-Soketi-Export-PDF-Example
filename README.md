# Soketi

The server is built on top of [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js) - a C application ported to Node.js. uWebSockets.js is demonstrated to perform at levels [8.5x that of Fastify](https://alexhultman.medium.com/serving-100k-requests-second-from-a-fanless-raspberry-pi-4-over-ethernet-fdd2c2e05a1e) and at least [10x that of Socket.IO](https://medium.com/swlh/100k-secure-websockets-with-raspberry-pi-4-1ba5d2127a23). ([source](https://github.com/uNetworking/uWebSockets.js))

## Installing with NPM

> Node.js LTS (14.x, 16.x, 18.x) is required due to uWebSockets.js build limitations.

Soketi may be easily installed via the NPM CLI:

```shell
npm install -g @soketi/soketi
```

If installation fails with error code 128 as shown, delete `/root/.npm` folder and try again.

```
npm ERR! code 128
npm ERR! An unknown git error occurred
npm ERR! command git --no-replace-objects clone -b v20.10.0 ssh://git@github.com/uNetworking/uWebSockets.js.git /root/.npm/_cacache/tmp/git-cloneOvhFm4 --recurse-submodules --depth=1
npm ERR! fatal: could not create leading directories of '/root/.npm/_cacache/tmp/git-cloneOvhFm4': Permission denied
```

After installation, a soketi server using the default configuration may be started using the start command:

```shell
soketi start
```
By default, this will start a server at `127.0.0.1:6001` with the following application credentials:

- App ID: `app-id`
- App Key: `app-key`
- Secret: `app-secret`

# Laravel Broadcasting

When using [Laravel's event broadcasting](https://laravel.com/docs/9.x/broadcasting) feature within your application, soketi is even easier to configure. First, replace the default pusher configuration in your application's config/broadcasting.php file with the following configuration:

```php
'connections' => [

    // ...

    'pusher' => [
        'driver' => 'pusher',
        'key' => env('PUSHER_APP_KEY', 'app-key'),
        'secret' => env('PUSHER_APP_SECRET', 'app-secret'),
        'app_id' => env('PUSHER_APP_ID', 'app-id'),
        'options' => [
            'host' => env('PUSHER_HOST', '127.0.0.1'),
            'port' => env('PUSHER_PORT', 6001),
            'scheme' => env('PUSHER_SCHEME', 'http'),
            'encrypted' => true,
            'useTLS' => env('PUSHER_SCHEME') === 'https',
        ],
    ],
],
```

To broadcast events using Pusher Channels, we need to install the Pusher Channels PHP SDK using the Composer package manager:

```shell
composer require pusher/pusher-php-server
```

# Laravel Echo

To configure client side we need to install Laravel Echo ant PusherJS libraries:

```shell
npm install laravel-echo pusher-js -D
```

Laravel Echo is compatible with the PusherJS library. Therefore, its configuration resembles the typical configuration of a PusherJS client such as the example configuration in the previous section of documentation:

```
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    wsHost: import.meta.env.VITE_PUSHER_HOST ?? `ws-${import.meta.env.VITE_PUSHER_APP_CLUSTER}.pusher.com`,
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 80,
    wssPort: import.meta.env.VITE_PUSHER_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_PUSHER_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss']
});
```

> Make sure that enabledTransports is set to ['ws', 'wss']. If not set, in case of connection failure, the client will try other transports such as XHR polling, which soketi doesn't support.

## Excluding event recipients

In some situations, you want to stop the client that broadcasts an event from receiving it.

Each pusher connection is assigned a unique `socket_id` which can be accessed via

```
channel.pusher.connection.socket_id
```

Once the `socket_id` has been accessed it can be used when triggering an event on the server by passing it to the server.

```
button.addEventListener('click', function () {
  axios.post('/api/my-action', {
    socket_id: channel.pusher.connection.socket_id
  });
});
```

When you trigger an event from the server passing a `socket_id`, the Channels connection (client) with that `socket_id` will be excluded from receiving the event.

```
$payload = [
    'socket' => $request->socket_id
];

broadcast(new MyEvent($payload))->toOthers();
```

MyEvent might look like this:

```
use Illuminate\Support\Arr;

public $socket;

public function __construct(array $payload)
{
    $this->socket = Arr::pull($payload, 'socket');
}
```
