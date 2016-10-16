But isn't Zend Framework that overengineered, resource-hungry, enterprisey,
configuration heavy pile of …

#### Note

- Pause

---

## Maybe

<h2 class="fragment">Maybe Not</h2>

#### Note

- Over time, we added more and more features, and used increasing numbers of
  design patterns, giving it a Java-esque appearance.
- But we also spent the last year and more working on separating out the
  components, reducing dependencies, and working specifically on performance
  and resusability.

---

## Zend Framework is a *component library*

<h4 class="fragment">that also provides an MVC framework</h4>

#### Note

- Shipping an MVC framework doesn't lock you into that paradigm.
- This talk will demonstrate a number of useful components you can use in any
  project you write, whether it's a Zend Framework project, or otherwise.

---

## What we'll look at

- Services and workflows
- Data structures
- Security features
- Handling user input
- Servers and web services
- PSR-7 features

#### Note

- Around 20 components in total.

---

## Services and workflows

#### Note

- There are certain features you end up using in just about every modern
  application: DI containers and event systems.

---

## zend-servicemanager

#### High-level features

- container-interop compatible <!-- .element: class="fragment" -->
- factories and aliasing <!-- .element: class="fragment" -->
- abstract factories <!-- .element: class="fragment" -->
- delegator factories <!-- .element: class="fragment" -->
- lazy services <!-- .element: class="fragment" -->

#### Note

- Abstract factories allow a common factory for classes with the same
  instantiation pattern.
- Delegator factories are like decorators for factories, allowing de-coupling of
  service customization from basic construction.
- Lazy services wrap the service in a proxy, delaying actual instantiation until
  the service is invoked.

---

## Abstract Factories

```php
public function canCreate(
    ContainerInterface $container,
    $requestedName
) {
    if (! $container->has(SomethingElse::class)
        || ! $container->has(Dependency::class)
        || ! class_exists($requestedName)
    ) {
        return false;
    }
    return true;
}
```

#### Note

- You could also check if the class implements a particular interface, or is
  named a specific way, etc.

---

## Abstract Factories (cont)

```php
public function __invoke(
    ContainerInterface $container,
    $requestedName,
    array $options = []
) {
    $service = new $requestedName(
        $container->get(SomethingElse::class)
    );
    $service->setSomeOtherDependency(
        $container->get(Dependency::class)
    );
    return $service;
}
```

#### Note

- The interesting bit about this part of the interface is that it's exactly the
  same as our `FactoryInterface`, which means you can explicitly map classes to
  abstract factories as you would normal factories!

---

## Delegator Factories

```php
public function __invoke(
    ContainerInterface $container,
    $name,
    callable $callback,
    array $options = null
) {
    $service = $callback();
    $service->attach($container->get(SomeListener::class);
    return $service;
}
```

#### Note

- If you wanted, you could do pre-conditions here.
- This is an ideal way to attach listeners, observers, adapters, etc. to
  instances, as the original factory can be kept purposely small and targetted.

---

## Lazy Services

<ul>
    <li class="fragment"><code>Zend\ServiceManager\Proxy\LazyServiceFactory</code>
        is a <em>delegator factory</em></li>
    <li class="fragment">Uses <code>ocramius/proxy-manager</code> to generate
        <em>proxy classes</em> for your service</li>
    <li class="fragment">Proxy class lazy-loads the requested service from the
        container on first invocation.</li>
    <li class="fragment">The proxy class shares the same inheritance as the
        original class</li>
</ul>

#### Note

- This is essentially black magic!
- The use case is when you have services that may only be used conditionally;
  wrapping them as lazy services ensures that any expensive initialization
  operations are delayed until the first usage of the class.

---

## zend-eventmanager

#### High-level features

- Implement subject/observer, Aspect Oriented designs, pubsub, etc. <!-- .element: class="fragment" -->
- Listener priorities <!-- .element: class="fragment" -->
- Short-circuiting <!-- .element: class="fragment" -->
- Shared listeners <!-- .element: class="fragment" -->
- Wildcard listeners <!-- .element: class="fragment" -->
- Lazy listeners <!-- .element: class="fragment" -->
- Listener aggregates <!-- .element: class="fragment" -->

---

## Basic usage

```php
$events = new EventManager();
$events->attach('event-name', $listener);

$events->trigger('event-name', $this, [ 'param' => 'value']);
$events->triggerEvent(new Event(
    'event-name',
    $this,
    ['param' => 'value']
));
```

#### Note

- main reason for having triggerEvent vs trigger is so that you can have custom
  event types and/or re-use an event instance between triggers.

---

## Listeners

```php
function (EventInterface $e) {
    $name = $e->getName();
    $target = $e->getTarget();
    $params = $e->getParams();
    $param = $e->getParam('param');

    // do something...
}
```

#### Note

- You can also define custom event types, which may have additional methods on
  them.

---

## Listener prioritization

```php
$events->attach('event-name', $listener1, -100); // executes last!
$events->attach('event-name', $listener2);
$events->attach('event-name', $listener3); // executes after 2
$events->attach('event-name', $listener4, 100); // executes first!
```

#### Note

- Many event proponents say order shouldn't matter.
- However, it can be useful for creating workflow engines!

---

## Short-circuiting

```php
$shortCircuit = function ($result) {
    return $result instanceof Response;
};

$results = $events->triggerEventUntil($shortCircuit, $event);

if ($results->stopped()) {
    return $results->last();
}

// Or:
$listener = function ($e) {
    $e->stopPropagation();
    return new Result();
};
```

#### Note

- The first puts the short circuit logic in control of the subject; they define
  when a returned result will lead to short-circuiting.
- The second allows a listener to indicate when an error condition has occurred
  from which it cannot recover.

---

## Shared Listeners

```php
$shared = new SharedEventManager();
$shared->attach(Something::class, 'do', $listener);

$something = new Something();
$events = new EventManager($shared, [Something::class]);
$events->trigger('do', $something, []);
// $listener is trigered!
```

#### Note

- Essentially, it's a way for you to attach listeners without direct access to
  the target event manager instance.
- Use *interface* names as identifiers, and this essentially allows you to
  listen to an event emitted by any implementation of that interface.
- It's of dubious nature when you have delegator factories!

---

## Wildcard listeners

```php
$events->attach('*', $listener);
$shared->attach('*', 'do', $listener);
$shared->attach(Something::class, '*', $listener);
$shared->attach('*', '*', $listener);
```

#### Note

- For those times when you want one listener on every event of a given instance,
  or all named events of any instance, etc.
- Probably shouldn't do this!

---

## Lazy listeners

```php
$events->attach('do', new LazyListener([
    'listener' => Listener::class,
    'method'   => 'do',
], $container);
```

#### Note

- Essentially, delays instantiation until the listener is triggered, at which
  point it will be pulled from the provided *container*.
- `$container` is a container-interop instance, such as zend-servicemanager.

---

## Listener aggregates

```php
class Listeners implements ListenerAggregateInterface
{
    use ListenerAggregateTrait;

    public function attach(EventManagerInterface $events, $priority = 1)
    {
        $events->attach('do', [$this, 'onDo'], $priority);
        $events->attach('something', [$this, 'onSomething'], $priority);
        $events->attach('else', [$this, 'onElse'], $priority);
    }
}
```

#### Note

- A way to define multiple related listeners in a single class.

---

## Data structures

#### Note

- We've seen some heavy hitters from ZF. Now let's look at some low-level
  features: code for working with data structures.

---

## zend-config

#### High-level features

- Read and parse INI, JSON, YAML, XML, and PHP configuration. <!-- .element: class="fragment" -->
- Write configuration to these formats. <!-- .element: class="fragment" -->
- OOP or array-like access to values. <!-- .element: class="fragment" -->

---

## Basic usage

```php
$config = \Zend\Config\Factory::fromFile('./config/config.yml');
$config = \Zend\Config\Factory::fromFile('./config/config.xml');
$config = \Zend\Config\Factory::fromFile('./config/config.json');
$config = \Zend\Config\Factory::fromFile('./config/config.php');
```

#### Note

- By default, returns a multi-dimensional array
- Passing a second boolean `true` argument casts it to a `Config` instance.

---

## Multiple files

```php
$config = \Zend\Config\Factory::fromFiles([
    './config/config.xml',
    './config/config.yaml',
    './config/config.json',
    './config/config.php',
], true);
```

#### Note

- Loads and parses each, and then *merges* with any previously loaded
  configuration!

---

## zend-dom

```php
use Zend\Dom\Query;

$dom = new Query($xmlOrHtml);
$results = $dom->execute('section.slide h2');

printf("Discovered %d items\n", count($results));

foreach ($results as $result) {
    printf("- %s\n", $result->nodeValue);
}
```

#### Note

- Use cases include screen scraping and end-to-end testing.
- Each item in the result set is a `DOMNode` instance.

---

## zend-feed

#### High-level features

- Fetch and parse RSS and Atom feeds. <!-- .element: class="fragment" -->
- Prepare and write RSS and Atom feeds. <!-- .element: class="fragment" -->

---

## zend-feed Reader: discovery

```php
use Zend\Feed\Reader\Reader;

$feedLinks = Reader::findFeedLinks($pageUrl);

foreach ($feedLinks as $link) {
    switch ($link['type']) {
        case 'application/atom+xml':
            $feedUrl = $link['href'];
            break;
        case 'application/rss+xml':
            $feedUrl = $link['href'];
            break;
    }
}
```

#### Note

- Sometimes we just want to know what's on a page.
- a `FeedSet` is returned, which is just an `ArrayObject`; each item it exposes
  is also a `FeedSet`, with specific attributes.

---

## zend-feed Reader: feeds

```php
use Zend\Feed\Reader\Reader;

$feed = Reader::import($feedUrl);

printf(
    "[%s](%s): %s\n",
    $feed->getTitle(),
    $feed->getLink(),
    $feed->getDescription()
);
```

#### Note

- You can also fetch the content and import it as a string.
- Uses zend-http by default, but you can provide your own client.
- You immediately have access to feed channel data

---

## zend-feed Reader: items

```php
foreach ($feed as $entry) {
    printf(
        "[%s](%s): %s\n",
        $entry->getTitle(),
        $entry->getLink(),
        $entry->getDescription()
    );
}
```

#### Note

- Similar to feeds, each entry gives you data.
- Supports feed extensions, which provide additional data in the feed and
  individual entries.

---

## zend-feed Writer

```php
use Zend\Feed\Writer\Feed;

$feed = new Feed();
$feed->setTitle('My l33t feed');
$feed->setLink('https://blog.example.com');
$feed->setFeedLink('https://blog.example.com/atom', 'atom');
$feed->setDateModified(time());

$entry = $feed->createEntry();
$entry->setTitle('First post!');
$entry->setLink('https://blog.example.com/01-first-post.html');
$entry->addAuthor(['name' => 'mwop', 'uri' => 'https://mwop.net']);
$entry->setDateCreated(time());
$entry->setDateModified(time());
$entry->setDescription($description);
$entry->setContent($content);
$feed->addEntry($entry);

echo $feed->export('atom');
```

#### Note

- You can build both atom and rss feeds.
- Quite a few other properties you can set; just like the reader, extensible via
  plugins.
- So, now the question is: how do we get a subset of items to represent in a
  feed?

---

## zend-paginator

```php
use Zend\Paginator\Adapter\Iterator as IteratorPaginator;
use Zend\Paginator\Paginator;

$paginator = new Paginator(new IteratorPaginator($resultSet));
$paginator->setCurrentPageNumber($page);
$paginator->setItemCountPerPage(15);

printf(
    "Page %d of %d (displaying %d items of %d total)\n",
    $paginator->getCurrentPageNumber(),
    (count) $paginator,
    $paginator->getItemCountPerPage(),
    $paginator->getTotalItemCount()
);

foreach ($paginator as $item) {
    // ...
}
```

#### Note

- Paginator is an iterator.
- It will only return as many items as you've specified as the item count per page.
- Count of the paginator itself is the number of *pages* of results.

---

## zend-paginator: included adapters

- `ArrayAdapter`
- `Callback`
- `DbSelect` (zend-db) 
- `DbTableGateway` (zend-db) 
- `Iterator`
- `NullFill`

#### Note

- The callback adapter requires two callbacks, one for retrieving the count of
  items, and another for retrieving the items.
- The NullFill adapter allows you to specify a number of items and returns
  arrays of null values.

---

## zend-paginator: roll your own adapter

```php
namespace Zend\Paginator\Adapter;

use Countable;

interface AdapterInterface extends Countable
{
    /**
     * @param int $offset
     * @param int $itemCountPerPage
     * @return array
     */
    public function getItems($offset, $itemCountPerPage);
}
```

---

## Security

#### Note

- Secure programming practices are a must. These range from validating user
  input, which I'll cover later, to escaping output, ensuring users are
  authorized to perform actions, hashing passwords, and even encryption.

---

## zend-crypt

#### High-level features

- Encryption and decryption algorithms, including encrypt-then-authenticate
  implementation. <!-- .element: class="fragment" -->
- Key exchange and derivation. <!-- .element: class="fragment" -->
- Secure password hashing. <!-- .element: class="fragment" -->
- Hash and HMAC value generation. <!-- .element: class="fragment" -->

#### Note

- I don't pretend to understand the majority of this.
- Our password hashing can be used as a polyfill for the PHP 7 hashing and
  verification algorithms; we use the PHP 7 functionality when running under PHP
  7.
- I only want to talk about one specific feature, however: end-to-end
  encryption.

---

## End-to-end Encryption

```php
use Zend\Crypt\Hybrid;

// Encryption:
$publicKey = file_get_contents('public.pem');
$hybrid = new Hybrid();
echo $hybrid->encrypt($message, $publicKey);

// Decryption
$privateKey = file_get_contents('secret.pem');
echo $hybrid->decrypt($encrypted, $privateKey);
```

#### Note

- Essentially, this allows you to store public keys for users, and encrypt data
  using those keys; the recipients can then decrypt the data using their own
  private key.
- Ideally, the private keys remain in the possession of the users; these might
  be in their client applications, for instance.
- If you want to learn more, attend Enrico's talk Friday at 9.

---

### zend-escaper

```php
use Zend\Escaper\Escaper;

$escaper = new Escaper('utf-8');

echo $escaper->escapeHtml($html);
echo $escaper->escapeHtmlAttr($htmlAttribute);
echo $escaper->escapeJs($javascript);
echo $escaper->escapeCss($css);
echo $escaper->escapeUrl($url);
```

#### Note

- There are different ways of escaping output based on the context.
- There's really only one way to do it correctly for each context, but the
  methodology in PHP is relatively complex, particularly if you properly take
  into account the character encoding.
- Used in a number of existing template engines.

---

## zend-permissions-acl

### **A**ccess **C**ontrol **L**ists

#### Note

- We now move to authorization, where there are two industry-standard ways to
    authorize users: ACLs and RBACs. We'll look at the first now.

---

## zend-permissions-acl

```php
use Zend\Permissions\Acl\Acl;
use Zend\Permissions\Acl\Role\GenericRole as Role;
use Zend\Permissions\Acl\Resource\GenericResource as Resource;

$acl = new Acl();

$acl->addRole(new Role('guest'))
    ->addRole(new Role('member'))
    ->addRole(new Role('admin'));

$parents = ['guest', 'member', 'admin'];
$acl->addRole(new Role('mwop'), $parents);

$acl->addResource(new Resource('blog'));

$acl->deny('guest', 'blog');
$acl->allow('member', 'blog');

echo $acl->isAllowed('mwop', 'blog') ? 'ALLOW' : 'DENY'; // ALLOW
echo $acl->isAllowed('guest', 'blog') ? 'ALLOW' : 'DENY'; // DENY
```

#### Note

- You can operate ACLs as whitelists or blacklists.
- Roles and resources can be defined as strings to simplify creation.
- Create custom roles and resources if you need custom capabilities.
- Roles and Resources are hierarchical
- No data storage mechanisms currently. However, these are relatively easy to
  create; zf-mvc-auth, from Apigility, shows a config-driven approach.

---

## zend-permissions-rbac

### **R**ole-**B**ased **A**ccess **C**ontrols

#### Note

- I like to think of RBAC as a *tagging* approach. Instead of a user having a
  specific role in the inheritance tree, the full set of permissions is based on
  the various roles they are assigned.
- Each *role* then is assigned permissions, which can be arbitrary. It's similar
  to ACL resources, but the idea is that if the permission is present, it's an
  *allowed* action; absence of the permission indicates *denial*.

---

## zend-permissions-rbac

```php
use Zend\Permissions\Rbac\Rbac;
use Zend\Permissions\Rbac\Role;

$rbac = new Rbac();
$foo  = new Role('foo');
$foo->addPermission('bar');

var_dump($foo->hasPermission('bar')); // true

$rbac->addRole($foo);
$rbac->isGranted('foo', 'bar'); // true
$rbac->isGranted('foo', 'baz'); // false
```

#### Note

- You essentially have two ways of testing permissions. If you have access to
  the Role instance, you can call `hasPermission()`. If you only have access to
  the RBAC, you use `isGranted()`, passing both the role and the permission.

---

## Handling user input

#### Note

- So, we've now covered permissions. Let's think about user input.

---

## zend-filter

```php
namespace Zend\Filter;

interface FilterInterface
{
    public function filter($value);
}

class FilterChain implements FilterInterface
{
    public function attach(callable|FilterInterface $filter);
    public function attachByName($filter);
}
```

#### Note

- The filter interface is optional; you can also use any PHP callable.
- Filters take a value, and spit out the filtered value. These can be of any type.
- Filter chains allow you to create complex filters by chaining multiple filters.
- AttachByName uses a composed plugin manager, allowing you to pass just the filter name.

---

## Filter chains

```php
use Zend\Filter;

$chain = new Filter\FilterChain();
$chain->attach(new Filter\StripNewlines());
$chain->attach(new Filter\StringTrim());
$chain->attach(new Filter\Word\UnderscoreToSeparator());
$chain->attach(new Filter\UpperCaseWords());

$value = "\nsome_config_value    \n";
echo $chain->filter($value); // "Some Config Value"
```

#### Note

- Filter chains can be as complex as you need them!

---

## zend-validator

```php
namespace Zend\Validator;

interface ValidatorInterface
{
    public function isValid($value) : bool;
	public function getMessages() : array;
}

class ValidatorChain implements ValidatorInterface
{
    public function addValidator(
		ValidatorInterface $validator,
		 bool $breakChainOnFailure = false
	);
    public function prependValidator(/* ... */);
    public function addByName(
        $name,
        $options = [],
        bool $breakChainOnFailure
    );
    public function prependByName(/* ... */);
}
```

#### Note

- `isValid()` tests a value for validity.
- `getMessages()` returns validation failure messages.
- Validator chains allow you to create complex validations out of smaller, more
  discrete validators.

---

## Validator chains

```php
use Zend\Validator;

$chain = new Validator\ValidatorChain();
$chain->addValidator(new Validator\NotEmpty(), true);
$chain->addValidator(new Validator\Digits());
$chain->addValidator(new Validator\Between([
	'min' => 100,
	'max' => 300,
]), true);
$chain->addValidator(new Validator\Step(3), true);

echo $chain->isValid(201) ? 'VALID : 'INVALID'; // VALID
echo $chain->isValid(98) ? 'VALID : 'INVALID'; // INVALID
echo "Failed validation:
foreach ($chain->getMessages() as $message) {
	printf("- %s:\n", $message);
}

// - The input is not between "100" and "300", inclusively.
// - The input is not a valid step.
```

#### Note

- We can skip running further validations if we didn't receive a value at all.
- We can have narrow scopes for each validation, and report all the ways in
  which it failed.
- Messages can be customized and localized!

---

## zend-inputfilter

#### High-level features

- Pre-filter, then validate, individual inputs. <!-- .element: class="fragment" -->
- Compose inputs as sets into an input filter. <!-- .element: class="fragment" -->
- Validate subsets via validation groups. <!-- .element: class="fragment" -->
- Validate collections. <!-- .element: class="fragment" -->

#### Note

- Essentially, an input composes a filter chain and a validator chain.
- Filters are executed prior to validators.
- Input sets might be a *form*, a *fieldset*, or a web service *resource*
- Think of a collection as a repeated fieldset.
- You can define everything programmatically, or via configuration-driven
  specifications.

---

## zend-inputfilter

```php
use Zend\Filter;
use Zend\InputFilter\InputFilter;
use Zend\InputFilter\Input;
use Zend\Validator;

$email = new Input('email');
$email->getFilterChain()
    ->attach(new Filter\StringTrim());
$email->getValidatorChain()
    ->addValidator(new Validator\EmailAddress());

$password = new Input('password');
$password->getValidatorChain()
    ->attach(new Validator\StringLength(8));

$inputFilter = new InputFilter();
$inputFilter->add($email);
$inputFilter->add($password);
$inputFilter->setData($request->getParsedBody());

echo $inputFilter->isValid() ? 'VALID' : 'INVALID';
```

#### Note

- All inputs default to "required", unless you specify otherwise.

---

## zend-inputfilter

```php
$values = $inputFilter->getValues(); // filtered, valid data!
$rawValues = $inputFilter->getRawValues(); // unfiltered data!

$messages = $inputFilter->getMessages();
/*  [
        'field' => [ /* messages for field */ ]
    ] */

$inputFilter->setValidationGroup(['email']);
// validate only 'email' input
```

#### Note

- Essentially, you have to work hard to get the raw or invalid values.
- Messages follow the same format as zend-validator, but with an additional
  level of nesting.

---

## zend-hydrator

#### High-level features

- Cast objects to arrays (extract). <!-- .element: class="fragment" -->
- Cast arrays to objects (hydrate). <!-- .element: class="fragment" -->
- Filter properties during extraction. <!-- .element: class="fragment" -->
- Strategies (e.g., converting values to booleans or DateTime). <!-- .element: class="fragment" -->
- Naming strategies (to transform keys/names during operations). <!-- .element: class="fragment" -->

#### Note

- So, we can now validate the incoming data; how do we then *consume* it?
- Essentially, with hydrators, we can map incoming input to objects, and
  business objects to output.

---

## zend-hydrator

```php
use Zend\Hydrator;

class User
{
    private $email;
    private $password;
}

$hydrator = new Hydrator\Reflection();
$hydrator->addFilter('stripPassword', function ($property) {
    return $property !== 'password';
});
$user = $hydrator->hydrate($inputFilter->getValues(), new User());

echo json_encode($hydrator->extract($user)); // {"email": "..."}
```

#### Note

- We choose an appropriate hydrator for our object.
- We filter properties; returning boolean true means we keep the property.
- Hydration does not trigger filters, so the full submitted data is provided to us.
- Extraction triggers filtering.

---

## Servers and web services

#### Note

- We're now starting to venture into the realm of frameworks.
- And yet... we're not. Traditional web service gateways often directly expose objects or
  functions.
- Zend Framework has several such servers in place, all following the same
  paradigms as PHP's own SoapServer.

---

### Servers offered

- zend-xmlrpc: XML-RPC server (and client!) <!-- .element: class="fragment" -->
- zend-soap: SOAP server (and client!) <!-- .element: class="fragment" -->
- zend-json-server: JSON-RPC server (and client!) <!-- .element: class="fragment" -->

#### Note

- zend-xmlrpc server was the first major contribution I made to the framework.
- We're only going to look at the server implementations
- all follow the same basic patterns.

---

### XML-RPC

```php
use Zend\XmlRpc\Server as XmlRpcServer;

$server = new XmlRpcServer();
$server->setClass(SomeService::class, 'some'); // some.*()
$server->addFunction('aFunction', 'a');        // a()
echo $server->handle();
```

#### Note

- Uses reflection, so it will look at docblocks to determine parameter names and
  expected values, as well as expected return values.
- Allows "namespacing" classes, and aliasing functions.
- Incorporates fault responses and system methods for introspection.

---

## SOAP: WSDL

```php
use Zend\Soap\AutoDiscover;

$uri = $request->getUri();
if ($request->getMethod() === 'GET') {
    $autodiscover = new AutoDiscover();
    $autodiscover->setClass(SomeService::class);
    $autodiscover->addFunction('aFunction');
    $autodiscover->setUri((string) $uri);

    $response->getBody()->write($autodiscover->toXml());
    
    return $response
        ->withHeader('Content-Type', 'application/wsdl+xml');
}
```

#### Note

- This is a little more involved, as we need to be able to both generate WSDL
  *and* serve the SOAP request.
- This example generates the WSDL when a GET request is made to the URI.

---

## SOAP: Server

```php
use Zend\Soap\Server as SoapServer;

if ($request->getMethod() !== 'POST') {
    return $response->withStatus(405);
}

$server = new SoapServer((string) $uri);
$server->setReturnResponse(true);
$server->setClass(SomeService::class);
$server->addFunction('aFunction');
$response->getBody()->write($server->handle());
return $response->withHeader('Content-Type', 'application/soap+xml');
```

#### Note

- The AutoDiscover and SoapServer implementations mirror each other, with minor
  differences around URI injection and behavior.
- This handles the request.
- You'll note that I'm using request and response objects. We'll see those in
  more detail later, but you could also do this using PHP's superglobals.

---

## zend-json-server: SMD

```php
use Zend\Json\Server\Server as JsonRpcServer;
use Zend\Json\Server\Smd;

$server = new JsonRpcServer();
$server->setClass(SomeService::class);
$server->addFunction('aFunction');
$server->setAutoEmitResponse(false);

$uri = $request->getUri();
if ($request->getMethod() === 'GET') {
    $server->setTarget((string) $uri);
    $server->setEnvelope(Smd::ENV_JSONRPC_2);
    $map = $server->getServiceMap();
    $response->getBody()->write((string) $map);
    return $response->withHeader('Content-Type', $map->getContentType());
}
```

#### Note

- JSON-RPC defines a "Service Map Description", which is essentially WSDL for
  JSON-RPC. So, like SOAP, GET requests will generate and return that.
- Unlike SOAP, however, we only need to define the server instance, as it
  composes the service map.

---

## zend-json-server: Server

```php
if ($request->getMethod() !== 'POST') {
    return $response->withStatus(405);
}

$response->getBody()->write((string) $server->handle());
return $response->withHeader(
    'Content-Type',
    $server->getServiceMap()->getContentType()
);
```

#### Note

- The service map contains the content type for responses.
- Otherwise, this should look remarkably similar to the previous servers!

---

## PSR-7

---

## HTTP Message Interfaces

- `StreamInterface` (message bodies)
- `MessageInterface` (compose streams, headers, protocol version)
- `ResponseInterface` (add status code and reason)
- `UriInterface` (value object describing URI)
- `RequestInterface` (compose URI and method)
- `UploadedFileInterface` (value object representing upload)
- `ServerRequestInterface` (compose server env, cookies, query params, parsed
  body params, uploaded files, and arbitrary attributes)

---

## zend-diactoros

- Developed along with the specification. <!-- .element: class="fragment" -->
- Complete PSR-7 implementation. <!-- .element: class="fragment" -->
- Factory for server requests. <!-- .element: class="fragment" -->
- Simplified response types. <!-- .element: class="fragment" -->
- Response emitters. <!-- .element: class="fragment" -->

#### Note

- Essentially the de facto standard PSR-7 implementation; used by Symfony's
  PSR-7 bridge, ZF itself, and several other PSR-7 middleware frameworks.

---

## Server requests

```php
use Zend\Diactoros\ServerRequestFactory;

$request = ServerRequestFactory::fromGlobals(); // use superglobals
$request = ServerRequestFactory::fromGlobals(
    $serverValues,    // $_SERVER
    $queryParameters, // $_GET
    $bodyParameters,  // $_POST
    $cookies,         // $_COOKIE
    $uploadedFiles    // $_FILES
);
```

#### Note

- At the application level, you can call it without arguments
- During testing, you can seed it using static values
- If you are using an async server, these values might be provided by the
  server, but not available in superglobals!

---

## Custom response types

- `EmptyResponse($status, array $headers)`
- `HtmlResponse($content, $status, array $headers)`
- `JsonResponse($data, $status, array $headers)`
- `RedirectResponse($uri, $status, array $headers)`
- `TextResponse($content, $status, array $headers)`

#### Note

- These exist to simplify the most common response types you might use.
- Usually an appropiate Content-Type is provided by default.
- Note the consistency of the constructors!

---

## Response emitters

```php
namespace Zend\Diactoros\Response;

use Psr\Http\Message\ResponseInterface;

interface EmitterInterface
{
    public function emit(ResponseInterface $response) : void;
}
```

- SapiEmitter <!-- .element: class="fragment" -->
- SapiStreamEmitter <!-- .element: class="fragment" -->

#### Note

- Emitters let you define exactly how a response will be emitted back to the
  client.
- Default SAPI emitter emits the status line, headers, and body.
- Stream Emitter honors Content-Range, and loops over the stream to emit it in
  chunks.

---

## zend-stratigility

### Middleware Foundation

Build PSR-7 middleware pipelines.

---

## Middleware

```php
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Message\ResponseInterface as Response;

function (
    Request $request,
    Response $response,
    callable $next
) : Response {
    // do some work

    // delegate to a deeper layer:
    $response = $next($request, $response);

    // do more work

    return $response;
}
```

#### Note

- That's really it.
- Don't use the response passed to you. Create one, or manipulate the one
  returned from calling `$next()`

---

## Pipelines

```php
use Zend\Stratigility\MiddlewarePipe;

$pipeline = new MiddlewarePipe();
$pipeline->pipe(new XPoweredBy());
$pipeline->pipe(new ErrorHandler());
$pipeline->pipe(new ContentNegotiation());
$pipeline->pipe(new RoutingMiddleware());
$pipeline->pipe(new DispatchRoutedMiddleware());
$pipeline->pipe(new NotFound());
```

---

## Segregated Pipelines

```php
$api = new ApiPipeline();

$pipeline->pipe('/api', $api);
```

#### Note

- Literal matches only! Don't use this for routing!
- Nested middleware, when executed, receives a request that strips the leading,
  matched part of the URI. This allows you to do routing without worrying about
  the base path.
- If the URI does not match, it just skips over it!

---

## Expressive

### PSR-7 Middleware in Minutes

- Container-based middleware; use container-interop. <!-- .element: class="fragment" -->
- Routing interfaces; use whatever you want. <!-- .element: class="fragment" -->
- Templating interfaces; use whatever you want. <!-- .element: class="fragment" -->
- Error handling provided, but use whatever you want. <!-- .element: class="fragment" -->
- Installer lets you choose what you want. <!-- .element: class="fragment" -->

---

## Expressive applications

```php
// Pipeline
$app = $container->get(\Zend\Expressive\Application::class);
$app->pipe(new XPoweredBy());
$app->pipe(new ErrorHandler());
$app->pipe(new ContentNegotiation());
$app->pipeRoutingMiddleware();
$app->pipeDispatchMiddleware();
$app->pipe(new NotFound());

// Routes
$app->get('/', HomePage::class, 'home');
$app->get('/contact', Contact::class, 'contact');
$app->post('/contact', ContactProcess::class);
$app->get('/users', UserList::class, 'users');
$app->post('/users', UserCreate::class);
$app->get('/users/{id:[a-z]+}', User::class, 'user');
$app->patch('/users/{id:[a-z]+}', UserUpdate::class);
$app->delete('/users/{id:[a-z]+}', UserDelete::class);
```

#### Note

- This can also be done via configuration
- Pipelines are for the entire application
- Routed middleware uses HTTP verbs

---

## Routed pipelines

```php
$create = \Zend\Expressive\AppFactory::create($container);
$create->pipe(Authentication::class);
$create->pipe(Authorization::class);
$create->pipe(ContentValidation::class);
$create->pipe(UserCreate::class);

$app->post('/users', $create);
```

---

## Resources

- https://docs.zendframework.com/

---

## Thank You!
