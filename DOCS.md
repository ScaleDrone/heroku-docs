[ScaleDrone](http://addons.heroku.com/scaledrone) is the easiest way of adding real-time capabilities to your app. You can also sign up and use the service from [www.scaledrone.com](https://www.scaledrone.com).

# Getting started

To include the ScaleDrone client library in your website, add a script tag to the `<head>` section of your HTML file.

```html
<script type='text/javascript' src='https://api2.scaledrone.com/assets/scaledrone.min.js'></script>
```

_Do not host this file yourself._

## Variables

Heroku gives you access to two variables:
* `SCALEDRONE_CHANNEL_ID` - this is your channels public ID
* `SCALEDRONE_CHANNEL_SECRET` - this is a secret key you use to create access tokens for clients and servers

You can access them as environment variables from Heroku or from the Heroku toolbelt with:
```term
$ heroku config:get SCALEDRONE_CHANNEL_ID
$ heroku config:get SCALEDRONE_CHANNEL_SECRET
```

## JavaScript Client

To subscribe to messages you can use the JavaScript API. If you are serving the client as a template you can replace `'CHANNEL_ID'` with `SCALEDRONE_CHANNEL_ID `enviroment variable.

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

## REST Pushing

The REST API lets you push data to rooms as HTTPS POST requests.

```POST
URL:
  https://api2.scaledrone.com/[channel_id]/[room_name]/publish
Data: {"hello": "from REST"}
```

**Headers:**

| Header        | Required | Value                        | Description |
| ------------- |:--------:| ---------------------------- | ----------- |
| Content-Type  | ✗        | application/json             | Set content type to JSON if you want to receive data as JavaScript objects |
| Authorization | ✗ you can also make unauthorized request      | Bearer eyJ0eXAiOiJKV1QiLC... | Read about the Authorization header at the [authentication documentation](/docs/authentication) |

**Response codes:**

| Code | Description |
| ---- | ----------- |
| 200  | Everything went OK, the message was published |
| 400+ | Error, response message gives a more detailed error message |

# Authentication

Both JavaScript and REST connections can be authenticated using [JSON Web Tokens](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html) (JWT). The token is encoded using your channel's secret.
There are JWT libraries for most programming languages and it is relatively easy to implement yourself.

## JSON Web Token

### JWT Header
[JSON Web Tokens](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html) (JWT) has to be encoded with HMAC using SHA-256 (HS256). The decoded header always looks like this:
```
{"alg": "HS256", "typ": "JWT"}

```
### JWT Payload
JWT's payload uses the common 'ext' JWT claim and some ScaleDrone's specific claims. An example decoded JWT payload looks like this:
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
| client      | ✗ (not required for REST API) | The client connection's ID provided by the JavaScript client after the 'open' event |
| channel     | ✔        | Channel's ID that the token is for |
| permissions | ✔        | A regular expression hash that defined permissions to publish and subscribe to rooms |
| exp         | ✔        | Unix timestamp expiration time after which the token will not be accepted for processing |

### Permissions

The permissions claim is used to define which rooms the authenticated user can subscribe or publish to. It is possbile to define very detailed permission rules using [regular expressions](http://regexone.com/).
ScaleDrone uses the popular Perl regular expressions syntax used by most popular programming languages.

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
This allows the user to publish and subscribe to all rooms (regex: /.*/) besides 'main-room' (regex: /^main-room$"/).

<div class="note note-warning">To match a room called 'main' the correct regex is '^main$' not 'main'.
Otherwise it will match any room that contains the string 'main'.
</div>

## JavaScript Authentication

After connecting to ScaleDrone and catching the 'open' event the client can authenticate with a JWT (that the user got from your web server). ClientId can can be accessed from drone.clientId.

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

## REST Authentication

REST requests are authenticated using a JSON Web Token set as a header's Bearer token: `Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...`

**POST request structure:**
```curl
POST

URL:
    https://api2.scaledrone.com/[channel_id]/[room_name]/publish
Data:
    {"hello": "from REST, now with Auth!"}
Headers:
    Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJjaGFubmVsIjaibTdMWGVXVWlnN1FtWE1xVyIsInB1Ymxpc2giOnRydWUsImV4cCI6MjgxNjQwMzkxMzI2MH0.LKkmfbbwSol-veSanJaIOEI1trlDU9LbfrOHuAEb0Vo
```

_Don't set a client claim for REST API's JWT._

# Read more

You can find more documentation and tutorials at [ScaleDrone's documentation](https://www.scaledrone.com/docs).

# Examples

* [HTML5 Desktop Notifications](https://github.com/ScaleDrone/html5-javascript-push-notifications)
* [cURL Push](http://runnable.com/VCuSWvXxNucz1na6/scaledrone-curl-push-example-for-shell-and-bash)
* [PHP Push](http://runnable.com/VDfTR_CSFyNpVv79/scaledrone-php-push-example)
* [PHP Authentication](https://github.com/ScaleDrone/scaledrone-php)
* [Node.js Authentication (using Express)](https://github.com/ScaleDrone/scaledrone-express-jwt-demo)
