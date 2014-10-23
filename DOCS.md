[ScaleDrone](http://addons.heroku.com/scaledrone) is the easiest way to add real-time pushing capabilities to your app. ScaleDrone offers JavaScript and REST clients, so it doesn't matter which language your app uses.

You can also sign up and use the service from [www.scaledrone.com](https://www.scaledrone.com).

## Provisioning the add-on

ScaleDrone can be attached to a Heroku application via the  CLI:

> callout
> A list of all plans available can be found [here](http://addons.heroku.com/scaledrone).

```term
$ heroku addons:add scaledrone
-----> Adding ScaleDrone to sharp-mountain-4005... done, v18 (free)
```

Once ScaleDrone has been added `SCALEDRONE_CHANNEL_ID` and `SCALEDRONE_CHANNEL_SECRET` settings will be available in the app configuration and will contain the channel's ID and channel's secret. This can be confirmed using the `heroku config:get` command.

```term
$ heroku config:get SCALEDRONE_CHANNEL_ID
YOUR_PUBLIC_CHANNEL_ID

$ heroku config:get SCALEDRONE_CHANNEL_SECRET
YOUR_PRIVATE_CHANNEL_SECRET
```

After installing ScaleDrone the application should be configured to fully integrate with the add-on.

## Local setup

### Environment setup

After provisioning the add-on it's necessary to locally replicate the config vars so your development environment can operate against the service.

> callout
> Though less portable it's also possible to set local environment variables using `export SCALEDRONE_CHANNEL_ID=value` and `export SCALEDRONE_CHANNEL_SECRET=value`.

Use [Foreman](config-vars#local-setup) to configure, run and manage process types specified in your app's [Procfile](procfile). Foreman reads configuration variables from an .env file. Use the following command to add the values retrieved from heroku config to `.env`.

```term
$ heroku config -s >> .env
$ more .env
```

> warning
> Credentials and other sensitive configuration values should not be committed to source-control. In Git exclude the .env file with: `echo .env >> .gitignore`.

## Using the API

### JavaScript API

To include the ScaleDrone client library in your website, add a script tag to the `<head>` section of your HTML file.

```html
<script type='text/javascript' src='https://api2.scaledrone.com/assets/scaledrone.min.js'></script>
```
>warning
>Do not host this file yourself! You will not be notified when this file changes.

To subscribe to messages you can use the JavaScript API. If you are serving the client as a template you should replace `'CHANNEL_ID'` with `SCALEDRONE_CHANNEL_ID` enviroment variable.

```javascript
var drone = new ScaleDrone('CHANNEL_ID');
drone.on('open', function (error) {
    if (error) return console.error(error);
    var room = drone.subscribe('main');
    room.on('open', function (error) {
        if (error) console.error(error);
    });
    room.on('data', function (data) {
        console.log(data);
    });
    drone.publish({
        room: 'main',
        message: 'hey!'
    });
});

drone.on('close', function (event) {
    console.log('Connection was closed', event);
});

drone.on('error', function (error) {
    console.error(error);
});
```

This code opens a new connection to ScaleDrone, subscribes to messages from the room `main` and then sends a message to the room `main` saying 'hey!'.

### REST API

The REST API lets you push data to rooms as HTTPS POST requests.

```POST
URL:
  https://api2.scaledrone.com/[channel_id]/[room_name]/publish
Data: {"hello": "from REST"}
```

**Headers:**

| Header        | Required | Value                        | Description |
| ------------- |:--------:| ---------------------------- | ----------- |
| Content-Type  | Not required        | application/json             | Set content type to JSON if you want to receive data as JavaScript objects |
| Authorization | Not required. _Unauthorized requests are permitted_      | Bearer eyJ0eXAiOiJKV1QiLC... | Read about the Authorization header in the [authentication documentation](#rest-authentication) |

**Response codes:**

| Code | Description |
| ---- | ----------- |
| 200  | Everything went OK. The message was published. |
| 400+ | Error. The response message gives a more detailed error message. |

### Authentication

By default, ScaleDrone apps don't require authentication. This can be turned on from ScaleDrone's dashboard that you can access from your app's addons page.

#### JSON Web Token

Both JavaScript and REST connections can be authenticated using [JSON Web Tokens](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html) (JWT). The token is encoded using your channel's secret.
There are JWT libraries for most programming languages and it is relatively easy to implement yourself.

##### JWT Header

[JSON Web Tokens](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html) (JWT) has to be encoded with HMAC using SHA-256 (HS256). The decoded header always looks like this:

```json
{"alg": "HS256", "typ": "JWT"}
```

##### JWT Payload
JWT's payload uses the common `exp` JWT claim and some of ScaleDrone's specific claims. An example decoded JWT payload looks like this:

```json
{
    "client": "CLIENT_CONNECTION_ID",
    "channel": "CHANNEL_ID",
    "permissions": {
        ".*": {
            "publish": true,
            "subscribe": true
        },
        "^main-room$": {
            "publish": false,
            "subscribe": false
        }
    },
    "exp": 1408639878000
}
```

<br>

| Claim       | Required | Description |
| ----------- |:--------:| ----------- |
| client      | Not required for REST API | The client connection's ID provided by the JavaScript client after the 'open' event |
| channel     | ✔        | Channel's ID that the token is for |
| permissions | ✔        | A regular expression hash that defines permissions of the connection |
| exp         | ✔        | Unix timestamp expiration time after which the token will not be accepted for processing |

#### Permissions

The permissions claim is used to define which rooms the authenticated user can subscribe or publish to. It is possbile to define very detailed permission rules using [regular expressions](http://regexone.com/).
ScaleDrone uses the popular Perl regular expressions syntax used by most programming languages.

**Example permissions:**

```json
"permissions": {
    ".*": {
        "publish": true,
        "subscribe": true
    },
    "^main-room$": {
        "publish": false,
        "subscribe": false
    }
}
```
This allows the user to publish and subscribe to all rooms (regex: `/.*/`) besides 'main-room' (regex: `/^main-room$"/`).

<div class="note note-warning">To match a room called 'main' the correct regex is `^main$` not `main`.
Otherwise it will match any room that contains the string 'main'.
</div>

### JavaScript Authentication

After connecting to ScaleDrone and receiving the 'open' event, the client can authenticate with a JWT (that the user got from your web server). `ClientId` can be accessed from `drone.clientId`.

```javascript
var drone = new ScaleDrone('CHANNEL_ID');

drone.on('open', function (error) {
    if (error) return console.error(error);
    loadJwt(drone.clientId, function (jwt) {
        drone.authenticate(jwt);
    });
});

drone.on('authenticate', function (error) {
    if (error) return console.error(error);
    // Client is now authenticated
});
```

### REST Authentication

REST requests are authenticated using a JSON Web Token set as a header's Bearer token: `Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...`

**POST request structure:**

```
POST

URL:
    https://api2.scaledrone.com/[channel_id]/[room_name]/publish
Data:
    {"hello": "from REST, now with Auth!"}
Headers:
    Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJjaGFubmVsIjaibTdMWGVXVWlnN1FtWE1xVyIsInB1Ymxpc2giOnRydWUsImV4cCI6MjgxNjQwMzkxMzI2MH0.LKkmfbbwSol-veSanJaIOEI1trlDU9LbfrOHuAEb0Vo
```

>warning
>Don't set a client claim for REST API's JWT.

## Examples

* [HTML5 Desktop Notifications](https://github.com/ScaleDrone/html5-javascript-push-notifications)
* [cURL Push](http://runnable.com/VCuSWvXxNucz1na6/scaledrone-curl-push-example-for-shell-and-bash)
* [PHP Push](http://runnable.com/VDfTR_CSFyNpVv79/scaledrone-php-push-example)
* [PHP Authentication](https://github.com/ScaleDrone/scaledrone-php)
* [Node.js Authentication (using Express)](https://github.com/ScaleDrone/scaledrone-express-jwt-demo)

## Dashboard

The ScaleDrone dashboard allows you to see the current amount of users, see the history of your of concurrent users, access and change channel's variables.

The dashboard can be accessed via the CLI:

```term
$ heroku addons:open scaledrone
Opening ScaleDrone for sharp-mountain-4005â€¦
```

or by visiting the [Heroku apps web interface](http://heroku.com/myapps) and selecting the application in question. Select ScaleDrone from the Add-ons menu.

## Migrating between plans

> note
> Application owners should carefully manage the migration timing to ensure proper application function during the migration process.

Use the `heroku addons:upgrade` command to migrate to a new plan.

```term
$ heroku addons:upgrade scaledrone:large
-----> Upgrading ScaleDrone:small to sharp-mountain-4005... done, v18 ($130/mo)
       Your plan has been updated to: scaledrone:large
```

## Removing the add-on

ScaleDrone can be removed via the CLI.

> warning
> This will destroy all associated data and cannot be undone!

```term
$ heroku addons:remove scaledrone
-----> Removing ScaleDrone from sharp-mountain-4005... done, v20 (free)
```

## Support

All ScaleDrone support and runtime issues should be submitted via on of the [Heroku Support channels](support-channels). Any non-support related issues or product feedback is welcome on the [ScaleDrone contact page](https://www.scaledrone.com/contact).

## Read more

You can find more documentation and tutorials in the [ScaleDrone documentation](https://www.scaledrone.com/docs).
