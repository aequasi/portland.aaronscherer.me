# Guzzle
[Aaron Scherer][personal-blog]
June 1st, 2013

---

You can access these slides at

<http://portland.aaronscherer.me>

And view the documentation at

<http://guzzlephp.org/>

Press the `Escape` key to see all the slides.

#

## Ugh, Curl

* So many options
* Not OOP
* Complex

## Example

```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_COOKIEJAR, "/tmp/cookieFileName");
curl_setopt($ch, CURLOPT_URL,"http://www.someaspsite.com/checkpwd.asp");
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_POSTFIELDS, "UserID=username&password=passwd");
ob_start();      // prevent any output
curl_exec ($ch); // execute the curl command
ob_end_clean();  // stop preventing output

curl_setopt($ch, CURLOPT_RETURNTRANSFER,1);
curl_setopt($ch, CURLOPT_COOKIEFILE, "/tmp/cookieFileName");
curl_setopt($ch, CURLOPT_URL,"http://www.someaspsite.com/list.asp");
$buf2 = curl_exec ($ch);

curl_close ($ch);
```

What is this even doing!?!?


# Introducting - Guzzle!

#

## Guzzle Introduction
* PHP HTTP client & Framework
* Key for building RESTful web services
* Wrapper of curl
* Persisten Connections & Parallel Requests
* Streams
* Service Descriptions
* Lots of Plugins

## Example

```php
use Guzzle\Http\Client;
// Create a client and provide a base URL
$client = new Client('https://api.github.com');
// Create a request with basic Auth
$request = $client->get('/user')->setAuth('aequasi', 'mypass');
// Send the request and get the response
$response = $request->send();
echo $response->getBody();
// >>> {"type":"User", ... }
echo $response->getHeader('Content-Length');
```

In all its OOP goodness

## Example - cont

Configuring a request is pretty easy too

```php
$client = new Client('https://api.twitter.com/{version}', array(
	'version'        => 'v1.1', // could be something like $container->getParameter( 'twitter.api.version' )
	'curl.options'   => array(CURLOPT_PROXY => 'tcp://localhost:80'),
	'request.params' => array('foo' => 'bar')
));
```

# Requests

The different requests are as simple as 

* `$client->get()`
* `$client->head()`
* `$client->delete()`
* `$client->post()`
* `$client->put()`
* `$client->patch()`


If these dont work, you can also do `$client->createRequest()`

#

## URI Templates

Really Cool Feature

```php
$client = new Guzzle\Http\Client('https://example.com/', array('a' => 'hi'));
```

Followed by

```php
$request = $client->get(array('/test{?a,b}', array('b' => 'there'));
```

This will end up with a curl to `https://example.com/test/?a=hi&b=there`

## Taking it a step further

```php
$request = $client->get(array('http://example.com{+path}{/segments}{?query,data*}', array(
    'path'     => '/foo/bar',
    'segments' => array('one', 'two'),
    'query'    => 'test',
    'data'     => array(
        'more' => 'value'
    )
)));
```

Will end up as `http://example.com/foo/bar/one/two?query=test&more=value`

# Parallel and Persistence

#

## Parallel Requests

Handy for things like ping-post

```php
use Guzzle\Common\Exception\MultiTransferException;

try {
    $responses = $client->send(array(
        $client->get('http://www.google.com/'),
        $client->head('http://www.google.com/'),
        $client->get('https://www.github.com/')
    ));
} catch (MultiTransferException $e) {

    echo "The following exceptions were encountered:\n";
    foreach ($e as $exception) {
        echo $exception->getMessage() . "\n";
    }

    echo "The following requests failed:\n";
    foreach ($e->getFailedRequests() as $request) {
        echo $request . "\n\n";
    }

    echo "The following requests succeeded:\n";
    foreach ($e->getSuccessfulRequests() as $request) {
        echo $request . "\n\n";
    }
}
```

Once all the requests are completed, you will be able to run through each of the `foreach` loops

## Persistent Connections

* Keep connections open, saving network request time
* Using the same client ensures faster connections
* Built into curl, not normally used
* Clients use it by default

# Events!

#

## Example

Allows you to attach events to requests. Only one event right now, but still cool

```php
use Guzzle\Common\Event;
use Guzzle\Http\Client;

$client = new Client();

// Add a listener that will echo out requests as they are created
$client->getEventDispatcher()->addListener('client.create_request', function (Event $e) {
	var_dump( $e );
});
```

# Service Definitions

#

## Service Builder

* Any class that extends `Guzzle\Service\Client` or implements `Guzzle\Service\ClientInterface`
* Must implement `static::factory()`, can be called directly
* Commonly called through `Guzzle\Service\Builder\ServiceBuilder`

```php
use Guzzle\Service\Builder\ServiceBuilder;

// Create a service builder and provide client configuration data
$builder = ServiceBuilder::factory('/path/to/client_config.json');

// Get the client from the service builder by name
$twitter = $builder->get('twitter');
```


*/path/to/client_config.json*

```
{
    "services": {
        "twitter": {
            "class": "mtdowling\TwitterClient",
            "params": {
                "consumer_key": "****",
                "consumer_secret": "****",
                "token": "****",
                "token_secret": "****"
            }
        }
    }
}
```

And your service is defined


# Commands

#

## Intro

Very simple

```php
$command = $twitter->getCommand( 'getMentions', [ 'count' => 5 ] );
$result = $command->excute();
// Can be retrieved later without re-executing
$result = $command->getResult();
// You can even gte the request and the response
$request = $command->getRequest();
$response = $command->getResponse();

// OR, you can use magic methods
$jsonData = $twitter->getMentions( [ 'count' => 5 ] );
```

The results will be a json object

## Defining Commands

Defining commands is a little long winded, but, its as simple as creating a json file with the functions you want.

```
{
    "name": "Twitter",
    "apiVersion": "1.1",
    "baseUrl": "https://api.twitter.com/1.1",
    "description": "Twitter REST API client",
    "operations": {
        "GetMentions": {
            "httpMethod": "GET",
            "uri": "statuses/mentions_timeline.json",
            "summary": "Returns the 20 most recent mentions for the authenticating user.",
            "responseClass": "GetMentionsOutput",
            "additionalParameters": {
                "location": "query"
            }
        }
    },
    "models": {
        "GetMentionsOutput": {
            "type": "object",
            "additionalProperties": {
                "location": "json"
            }
        }
    }
}
```

And then, in your factory, setting the service description to that json file

## Parallel Commands

Commands can also be run in parallel by passing an array of commands to execute.

```php
use Guzzle\Service\Exception\CommandTransferException;

$commands = array();
$commands[] = $twitter->getCommand('getMentions');
$commands[] = $twitter->getCommand('otherCommandName');
// etc...

try {
    $result = $client->execute($commands);
    foreach ($result as $command) {
        echo $command->getName() . ': ' . $command->getResponse()->getStatusCode() . "\n";
    }
} catch (CommandTransferException $e) {
    // Get an array of the commands that succeeded
    foreach ($e->getSuccessfulCommands() as $command) {
        echo $command->getName() . " succeeded\n";
    }
    // Get an array of the commands that failed
    foreach ($e->getFailedCommands() as $command) {
        echo $command->getName() . " failed\n";
    }
}
```

## Events

Theres a few more events you can use here too. Documentation at 

<http://guzzlephp.org/webservice-client/webservice-client.html#plugins-and-events>


# Fin

[guzzle]: http://guzzlephp.org/
[personal-blog]: http://aaronscherer.me
