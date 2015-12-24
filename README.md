# psr7-middlewares


[![Build Status](https://travis-ci.org/oscarotero/psr7-middlewares.svg)](https://travis-ci.org/oscarotero/psr7-middlewares)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/oscarotero/psr7-middlewares/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/oscarotero/psr7-middlewares/?branch=master)

Collection of [PSR-7](http://www.php-fig.org/psr/psr-7/) middlewares.

It is installable and autoloadable via Composer as oscarotero/psr7-middlewares.

## Requirements

* PHP >= 5.5
* A PSR-7 HTTP Message implementation, for example [zend-diactoros](https://github.com/zendframework/zend-diactoros)
* A PSR-7 middleware dispatcher. For example [Relay](https://github.com/relayphp/Relay.Relay) or any other compatible with the following signature:

```php
use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

function (RequestInterface $request, ResponseInterface $response, callable $next) {
    // ...
}
```

## Usage example:

```php
use Psr7Middlewares\Middleware;

use Relay\RelayBuilder;
use Zend\Diactoros\Response;
use Zend\Diactoros\ServerRequestFactory;
use Zend\Diactoros\Stream;

//Set a stream factory used by some middlewares
//(Required only if Zend\Diactoros\Stream is not detected)
Middleware::setStreamFactory(function ($file, $mode) {
    return new Stream($file, $mode);
});

//Create a relay dispatcher and add some middlewares:
$relay = new RelayBuilder();

$dispatcher = $relay->newInstance([

    //Calculate the response time
    Middleware::responseTime(),

    //Add an Uuid to request
    Middleware::uuid(),
    
    //Handle errors
    Middleware::errorHandler('error_handler_function')->catchExceptions(true),

    //Override the method using X-Http-Method-Override header
    Middleware::methodOverride(),

    //Removes www subdomain
    Middleware::www(false)->redirect(301),

    //Block search engines robots indexing
    Middleware::robots(),

    //Add Google Analytics
    Middleware::googleAnalytics('UA-XXXXX-X'),

    //Geolocation
    Middleware::geolocate(),

    //Detect client device
    Middleware::detectDevice(),

    //Parse the request payload
    Middleware::payload(),

    //Remove the path prefix
    Middleware::basePath('/my-site/web'),

    //Remove the trailing slash
    Middleware::trailingSlash(),

    //Digest authentication
    Middleware::digestAuthentication(['username' => 'password']),

    //Get the client ip
    Middleware::clientIp(),

    //Allow only some ips
    Middleware::firewall(['127.0.0.*']),

    //Detects the user preferred language
    Middleware::languageNegotiator(['gl', 'es', 'en']),

    //Detects the format
    Middleware::formatNegotiator(),

    //Adds the php debug bar
    Middleware::debugBar($app->get('debugbar')),

    //Execute fast route
    Middleware::fastRoute($app->get('dispatcher')),

    //Minify the result
    Middleware::minify()

    //Saves the response in a file
    Middleware::saveResponse('app/public')
]);

$response = $dispatcher(ServerRequestFactory::fromGlobals(), new Response());
```

## Available middlewares

* [AccessLog](#accesslog)
* [AuraRouter](#aurarouter)
* [AuraSession](#aurasession)
* [BasePath](#basepath)
* [BasicAuthentication](#basicauthentication)
* [Cache](#cache)
* [ClientIp](#clientip)
* [Cors](#cors)
* [DebugBar](#debugbar)
* [DetectDevice](#detectdevice)
* [DigestAuthentication](#digestauthentication)
* [ErrorHandler](#errorhandler)
* [FastRoute](#fastroute)
* [FormTimestamp](#formtimestamp)
* [Firewall](#firewall)
* [FormatNegotiation](#formatnegotiation)
* [Geolocate](#geolocate)
* [GoogleAnalytics](#googleanalytics)
* [Honeypot](#honeypot)
* [ImageTransformer](#imagetransformer)
* [LanguageNegotiation](#languagenegotiation)
* [LeagueRoute](#leagueroute)
* [MethodOverride](#methodoverride)
* [Minify](#minify)
* [Payload](#payload)
* [Piwik](#piwik)
* [ReadResponse](#readresponse)
* [Rename](#rename)
* [ResponseTime](#responseTime)
* [Robots](#robots)
* [SaveResponse](#saveresponse)
* [Shutdown](#shutdown)
* [TrailingSlash](#trailingslash)
* [Uuid](#uuid)
* [Whoops](#whoops)
* [Www](#www)

### AuraRouter

To use [Aura.Router (3.x)](https://github.com/auraphp/Aura.Router) as a middleware:

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\AuraRouter;
use Aura\Router\RouterContainer;

//Create the router
$routerContainer = new RouterContainer();

$map = $routerContainer->getMap();

$map->get('hello', '/hello/{name}', function ($request, $response, $myApp) {

    //The route parameters are stored as attributes
    $name = $request->getAttribute('name');

    //You can get also the route instance
    $route = AuraRouter::getRoute($request);

    //Write directly in the response's body
    $response->getBody()->write('Hello '.$name);

    //or echo the output (it will be captured and writted into body)
    echo 'Hello world';

    //or return a string
    return 'Hello world';

    //or return a new response
    return $response->withStatus(200);
});

//Add to the dispatcher
$dispatcher = $relay->getInstance([

    Middleware::AuraRouter()
        ->router($routerContainer)   //Instance of Aura\Router\RouterContainer
        ->arguments($myApp)          //(optional) append more arguments to the controller
]);
```

### AccessLog

To generate access logs for each request using the [Apache's access log format](https://httpd.apache.org/docs/2.4/logs.html#accesslog). This middleware requires a [Psr log implementation](https://packagist.org/providers/psr/log-implementation), for example [monolog](https://github.com/Seldaek/monolog):

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\AuraRouter;
use Monolog\Logger;
use Monolog\Handler\ErrorLogHandler;

//Create the logger
$logger = new Logger('access');
$logger->pushHandler(new ErrorLogHandler());

//Add to the dispatcher
$dispatcher = $relay->getInstance([

    Middleware::ClientIp(), //Required to get the Ip

    Middleware::AccessLog()
        ->logger($logger) //Instance of Psr\Log\LoggerInterface
        ->combined(true)  //(optional) To use the Combined Log Format instead the Common Log Format
]);
```

### AuraSession

Creates a new [Aura.Session](https://github.com/auraphp/Aura.Session) instance with the request.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\AuraSession;

$dispatcher = $relay->getInstance([

    Middleware::AuraSession(),
        ->factory($sessionFactory) //(optional) Intance of Aura\Session\SessionFactory
        ->name('my-session-name'), //(optional) custom session name

    function ($request, $reponse, $next) {
        //Get the session instance
        $session = AuraSession::getSession($request);
    }
]);
```

### BasePath

Strip off the prefix from the uri path of the request. This is useful to combine with routers if the root of the website is in a subdirectory. For example, if the root of your website is `/web/public`, a request with the uri `/web/public/post/34` will be converted to `/post/34`.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    Middleware::BasePath('/web/public'),
]);
```

### BasicAuthentication

Implements the [basic http authentication](http://php.net/manual/en/features.http-auth.php). You have to provide an array with all users and password:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::BasicAuthentication()
        ->users([
            'username1' => 'password1',
            'username2' => 'password2'
        ])
        ->realm('My realm') //(optional) change the realm value
]);
```

### Cache

To save and reuse responses based in the Cache-Control: max-age directive and Expires header. You need a cache library compatible with psr-6

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::Cache()
        ->cache(new Psr6CachePool()) //the psr-6 cache implementation

    function($request, $response, $next) {
        //Cache the response 1 hour
        return $response->withHeader('Cache-Control', 'max-age=3600');
    }
]);
```

### ClientIp

Detects the client ip(s).

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\ClientIp;

$dispatcher = $relay->getInstance([

    Middleware::ClientIp()
        ->headers([
            'Client-Ip',
            'X-Forwarded-For',
            'X-Forwarded'
        ]), //(optional) to change the trusted headers

    function ($request, $response, $next) {
        //Get the user ip
        $ip = ClientIp::getIp($request);

        //Get all ips found in the headers
        $all_ips = ClientIp::getIps($request);

        return $next($request, $response);
    }
]);
```

### Cors

To use the [neomerx/cors-psr7](https://github.com/neomerx/cors-psr7) library:

```php
use Neomerx\Cors\Strategies\Settings

$relay = new RelayBuilder();

$settings = (new Settings())
    ->setServerOrigin([
        'scheme' => 'http',
        'host'   => 'example.com',
        'port'   => '123',
    ]);

$dispatcher = $relay->getInstance([

    Middleware::Cors()
        ->settings($settings)
]);
```

### DebugBar

Inserts the [PHP debug bar](http://phpdebugbar.com/) in the html body. This middleware requires `Middleware::formatNegotiator` executed before, to insert the debug bar only in Html responses.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::FormatNegotiator(),

    Middleware::DebugBar()
        ->debugBar(new DebugBar\StandardDebugBar())
]);
```

### DetectDevice

Uses [Mobile-Detect](https://github.com/serbanghita/Mobile-Detect) library to detect the client device.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\DetectDevice;

$dispatcher = $relay->getInstance([

    Middleware::DetectDevice(),

    function ($request, $response, $next) {
        //Get the device info
        $device = DetectDevice::getDevice($request);

        if ($device->isMobile()) {
            //mobile stuff
        }
        elseif ($device->isTablet()) {
            //tablet stuff
        }
        elseif ($device->is('bot')) {
            //bot stuff
        }

        return $next($request, $response);
    },
]);
```

### DigestAuthentication

Implements the [digest http authentication](http://php.net/manual/en/features.http-auth.php). You have to provide an array with the users and password:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::DigestAuthentication()
        ->users([
            'username1' => 'password1',
            'username2' => 'password2'
        ])
        ->realm('My realm') //(optional) custom realm value
        ->nonce(uniqid())   //(optional) custom nonce value
]);
```

### ErrorHandler

Executes a handler if the response returned by the next middlewares has any error (status code 400-599). You can catch also the exceptions throwed.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\ErrorHandler;

function errorHandler($request, $response, $myApp) {
    switch ($response->getStatusCode()) {
        case 404:
            return 'Page not found';

        case 500:
            //you can get the exception catched
            $exception = ErrorHandler::getException($request);

            return 'Server error: '.$exception->getMessage();

        default:
            return 'There was an error'
    }
}

$dispatcher = $relay->getInstance([

    Middleware::ErrorHandler()
        ->handler('errorHandler') //The error handler
        ->arguments($myApp)       //(optional) extra arguments to the handler
        ->catchExceptions()       //(optional) to catch exceptions if you don't use an external library for that
]);
```

### FastRoute
To use [FastRoute](https://github.com/nikic/FastRoute) as a middleware.

```php
use Psr7Middlewares\Middleware;

$router = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {

    $r->addRoute('GET', '/blog/{id:[0-9]+}', function ($request, $response, $app) {
        return 'This is the post number'.$request->getAttribute('id');
    });
});

$dispatcher = $relay->getInstance([

    Middleware::FastRoute()
        ->router($router)  //Instance of FastRoute\Dispatcher
        ->argument($myApp) //(optional) arguments appended to the controller
]);
```

### Firewall

Uses [M6Web/Firewall](https://github.com/M6Web/Firewall) to provide a IP filtering. This middleware deppends of **ClientIp** (to extract the ips from the headers).

[See the ip formats allowed](https://github.com/M6Web/Firewall#entries-formats) for trusted/untrusted options:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    //required to capture the user ips before
    Middleware::ClientIp(),

    //set the firewall
    Middleware::Firewall()
        ->trusted(['123.0.0.*'])   //(optional) ips allowed
        ->untrusted(['123.0.0.1']) //(optional) ips not allowed
]);
```

### FormatNegotiation

Uses [willdurand/Negotiation (2.x)](https://github.com/willdurand/Negotiation) to detect and negotiate the format of the document using the url extension and/or the `Accept` http header. It also adds the `Content-Type` header to the response if it's missing.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\FormatNegotiator;

$dispatcher = $relay->getInstance([

    Middleware::FormatNegotiator()
        ->defaultFormat('html') //(optional) default format if it's unable to detect. (by default is "html")
        ->addFormat('pdf', ['application/pdf', 'application/x-download']), //(optional) add new formats and mimetypes

    function ($request, $response, $next) {
        //get the format (for example: html)
        $format = FormatNegotiator::getFormat($request);

        return $next($request, $response);
    }
]);
```

### FormTimestamp

Simple spam protection based in injecting a hidden input in all post forms with the current timestamp. On submit the form, check the time value. If it's less than (for example) 3 seconds ago, assumes it's a bot, so returns a 403 response. You can also set a max number of seconds before the form expires.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    //required to get the format of the request (only executed in html requests)
    Middleware::FormatNegotiator(),

    Middleware::FormTimestamp()
        ->min(5)                  //(optional) Minimum seconds needed to validate the request (default: 3)
        ->max(3600)               //(optional) Life of the form in second. Default is 0 (no limit)
        ->inputName('time-token') //(optional) Name of the input (default: hpt_time)
        ->key('my-secret-key'),   //(optional but recomended) Key used to encrypt/decrypt the input value. If it's not defined, the value wont be encrypted
]);
```

### Geolocate

Uses [Geocoder library](https://github.com/geocoder-php/Geocoder) to geolocate the client using the ip. This middleware deppends of **ClientIp** (to extract the ips from the headers).

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\Geolocate;

$dispatcher = $relay->getInstance([
    
    //required to capture the user ips before
    Middleware::ClientIp(),

    Middleware::Geolocate()
        ->geocoder($geocoder), //(optional) To provide a custom Geocoder instance

    function ($request, $response, $next) {
        //get the location
        $addresses = Geolocate::getLocation($request);

        //get the country
        $country = $addresses->first()->getCountry();

        $response->getBody()->write('Hello to '.$country);

        return $next($request, $response);
    }
]);
```

### GoogleAnalytics

Inject the Google Analytics code in all html pages.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    //required to get the format of the request
    Middleware::formatNegotiator(),
    
    Middleware::GoogleAnalytics()
        ->id('UA-XXXXX-X') // The site id
]);
```

### Honeypot

Implements a honeypot spam prevention. This technique is based on creating a input field that should be invisible and left empty by real users but filled by most spam bots. The middleware scans the html code and inserts this inputs in all post forms and check in the incoming requests whether this value exists and is empty (is a real user) or doesn't exist or has a value (is a bot) returning a 403 response.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    //required to get the format of the request (only executed in html requests)
    Middleware::formatNegotiator(),
    
    Middleware::Honeypot()
        ->inputName('my_name') //(optional) The name of the input field (by default "hpt_name")
        ->inputClass('hidden') //(optional) The class of the input field (by default "hpt_input")
]);
```

### ImageTransformer

Uses [imagecow/imagecow](https://github.com/oscarotero/imagecow) to transform the images on demand. You can resize, crop, rotate and convert to other format. You can specify a predefined sizes using [the imagecow syntax](https://github.com/oscarotero/imagecow#execute-multiple-functions).

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    //required to get the format of the request
    Middleware::formatNegotiator(),

    Middleware::imageTransformer()
        ->basePath('/imgs')          // (optional) The base path of the images urls

    //Used to read the image files and returns the response with them
    Middleware::readResponse()
        ->storage('/path/to/images'),
]);
```

To resize or crop images on demand, use the following syntax: `[directory]/[transform].[filename]`. For example, to resize and crop the image `avatars/users.png` to 50x50px, the path is: `avatars/resizeCrop,50,50.user.png`. Because this method allows to generate unlimited images using random values, you can specify a list of named transform values:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    Middleware::formatNegotiator(),

    Middleware::imageTransformer()
        ->sizes([
            'small' => 'resizeCrop,50,50',
            'medium' => 'resize,500|format,jpg',
            'large' => 'resize,1000|format,jpg',
        ]), //(optional) The predefined sizes of the images.

    Middleware::readResponse('/path/to/images')
]);
```

Now, to get the 50x50 thumb, you have to use `avatars/small.user.png`. Any other value different to these predefined sizes returns a 404 response.

### LanguageNegotiation

Uses [willdurand/Negotiation](https://github.com/willdurand/Negotiation) to detect and negotiate the client language. You must provide an array with all available languages:

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\LanguageNegotiator;

$dispatcher = $relay->getInstance([

    Middleware::LanguageNegotiator()
        ->languages(['gl', 'en', 'es']), //Available languages

    function ($request, $response, $next) {
        //Get the preferred language
        $language = LanguageNegotiator::getLanguage($request);

        return $next($request, $response);
    }
]);
```

### LeagueRoute

To use [league/route (2.x)](https://github.com/thephpleague/route) as a middleware:

```php
use Psr7Middlewares\Middleware;
use League\Route\RouteCollection;

$router = new RouteCollection();

$router->get('/blog/{id:[0-9]+}', function ($request, $response, $vars) {
    return 'This is the post number'.$vars['id'];
});

$dispatcher = $relay->getInstance([

    Middleware::LeagueRoute()
        ->router($router) //The RouteCollection instance
]);
```

### MethodOverride

Overrides the request method using the `X-Http-Method-Override` header. This is useful for clients unable to send other methods than GET and POST:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::MethodOverride()
        ->get(['HEAD', 'CONNECT', 'TRACE', 'OPTIONS']), //(optional) to customize the allowed GET overrided methods
        ->post(['PATCH', 'PUT', 'DELETE', 'COPY', 'LOCK', 'UNLOCK']), //(optional) to customize the allowed POST overrided methods
]);
```

### Minify

Uses [mrclay/minify](https://github.com/mrclay/minify) to minify the html, css and js code from the responses.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    //needed to get the format of the response
    Middleware::formatNegotiator(),

    Middleware::Minify()
        ->forCache(true)   //(optional) only minify cacheable responses
        ->inlineCss(false) //(optional) enable/disable inline css minification
        ->inlineJs(false)  //(optional) enable/disable inline js minification
]);
```

### Payload

Parses the body of the request if it's not parsed and the method is POST, PUT or DELETE. It has support for json, csv and url encoded format.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    Middleware::Payload()
        ->associative(true) //(optional) To generate associative arrays with json objects

    function ($request, $response, $next) {
        //Get the parsed body
        $content = $request->getParsedBody();

        return $next($request, $response);
    }
]);
```

### Piwik

To use the [Piwik](https://piwik.org/) analytics platform. Injects the javascript code just before the `</body>` closing tag.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    //required to get the format of the request
    Middleware::formatNegotiator(),
    
    Middleware::Piwik()
        ->piwikUrl('//example.com/piwik')    // The url of the installed piwik
        ->siteId(1)                          // (optional) The site id (1 by default)
        ->addOption('setDoNotTrack', 'true') // (optional) Add more options to piwik API
]
```

### ReadResponse

Read the response content from a file. It's the opposite of [SaveResponse](#saveresponse)

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::ReadResponse()
        ->storage('path/to/document/root') //Path where the files are stored
        ->basePath('public')               //(optional) basepath ignored from the request uri
]);
```

### Rename

Renames the request path. This is useful in some use cases:

* To rename public paths with random suffixes for security reasons, for example the path `/admin` to a more unpredictible `/admin-19640983`
* Create pretty urls without use any router. For example to access to the path `/static-pages/about-me.php` under the more friendly `/about-me`

Note that the original path wont be publicly accesible. On above examples, requests to `/admin` or `/static-pages/about-me.php` returns 404 responses.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::Rename()
        ->paths([
            '/admin' => '/admin-19640983',
        ]),

    function ($request, $response, $next) {
        $path = $request->getUri()->getPath(); // /admin

        return $next($request, $response);
    }
]);
```

### ResponseTime

Calculates the response time (in miliseconds) and saves it into `X-Response-Time` header:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::ResponseTime()
]);
```

### Robots

Disables the robots of the search engines for non-production environment. Adds automatically the header `X-Robots-Tag: noindex, nofollow, noarchive` in all responses and returns a default body for `/robots.txt` request.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::Robots()
]);
```

### SaveResponse

Saves the response content into a file if all of the following conditions are met:

* The method is `GET`
* The status code is `200`
* The `Cache-Control` header does not contain `no-cache` value
* The request has not query parameters.

This is useful for cache purposes

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::SaveResponse()
        ->storage('path/to/document/root') //Path directory where save the responses
        ->basePath('public')               //(optional) basepath ignored from the request uri
]);
```

### Shutdown

Useful to display a 503 maintenance page. You need to specify a handler.

```php
use Psr7Middlewares\Middleware;

function shutdownHandler ($request, $response, $app) {
    $response->getBody()->write('Service unavailable');
}

$dispatcher = $relay->getInstance([

    Middleware::Shutdown()
        ->handler('shutdownHandler') //Callable that generate the response
        ->arguments($app)            //(optional) to add extra arguments to the handler
]);
```

### TrailingSlash

Removes (or adds) the trailing slash of the path. For example, `/post/23/` will be converted to `/post/23`. If the path is `/` it won't be converted. Useful if you have problems with the router.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::TrailingSlash()
        ->addSlash(true)     //(optional) to add the trailing slash instead remove
        ->redirect(301)      //(optional) to return a 301 (seo friendly) or 302 response to the new path
        ->basePath('public') //(optional) basepath
]);
```

### Uuid

Uses [ramsey/uuid](https://github.com/ramsey/uuid) to generate an Uuid (Universally Unique Identifiers) for each request (compatible with [RFC 4122](http://tools.ietf.org/html/rfc4122) versions 1, 3, 4 and 5). It's usefull for debugging purposes.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\Uuid;

$dispatcher = $relay->getInstance([

    Middleware::Uuid()
        ->version(4)     //(optional) version of the identifier (1 by default). Versions 3 and 5 need more arguments (see https://github.com/ramsey/uuid#examples)
        ->header(false), //(optional) Name of the header to store the identifier (X-Uuid by default). Set false to don't save header

    function ($request, $response, $next) {
        //Get the X-Uuid header
        $id = $request->getHeaderLine('X-Uuid');

        //Get the Uuid instance
        $uuid = Uuid::getUuid($request);

        echo $uuid->toString();

        return $next($request, $response);
    }
]);
```

### Whoops

To use [whoops](https://github.com/filp/whoops) as error handler.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\Whoops;
use Whoops\Run;

$dispatcher = $relay->getInstance([

    Middleware::Whoops()
        ->whoops(new Run())  //(optional) provide a whoops instance
        ->catchErrors(false) //(optional) to catch not only exceptions but also php errors (true by default)
]);
```

### Www

Adds or removes the `www` subdomain in the host uri and, optionally, returns a redirect response. The following types of host values wont be changed:
* The one word hosts, for example: `http://localhost`.
* The ip based hosts, for example: `http://0.0.0.0`.
* The multi domain hosts, for example: `http://subdomain.example.com`.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::Www()
        ->addWww(true)  //(optional) Add www instead remove it
        ->redirect(301) //(optional) to return a 301 (seo friendly) or 302 response to the new host
]);
```


## Lazy/conditional middleware creation

You may want to create middleware in a lazy way under some circunstances:

* The middleware is needed only in a specific context (for example in development mode)
* The middleware creation is expensive and is not needed always (because a previous middleware returns a cached response)

To handle with this, you can use the `Middleware::create()` method that must return a callable or false. Example:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    //This middleware can return a cached response
    //so the next middleware may not be executed
    Middleware::cache($myPsr6CachePool),

    //Let's say this middleware is expensive, so use a proxy for lazy creation
    Middleware::create(function () use ($app) {
        return Middleware::auraRouter($app->get('router'));
    }),

    //This middleware is needed only in production
    Middleware::create(function () {
        return (getenv('ENV') !== 'production') ? false : Middleware::minify();
    })
]);
```


## Contribution

New middlewares are appreciated. Just create a pull request.
