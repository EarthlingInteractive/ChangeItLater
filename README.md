# The Phrebar Framework

Phrebar is the PHP framework used by
[PHPTemplateProject](http://github.com/EarthlingInteractive/PHPTemplateProject)
and any project based on it.

This repository documents some of the philosophy behind it.
More detailed technical documentation can be found in PHPTemplateProject itself,
or in the repositories of libraries that it uses.


## tl;dr

A web framwork that says
"I think you should do it like X,
but if you want to do it differently I won't get in your way"

Prerequisites: PHP 5.4, Composer, Postgres.

```sh
git clone https://github.com/EarthlingInteractive/PHPProjectInitializer.git
PHPProjectInitializer/bin/create-project -i --make everything
```


## Design goals

- Keep the application developer in control by:
  - Minimizing 'magic' and attempting to 'keep all the wires exposed'
  - Attempting to balance re-usable code (which goes into libraries)
    with stuff that the application developer might want to change
    (which gets copied in as boilerplate)
  - Holding 'frameworky' code to the same standards as library could world be,
    such as namespacing and avoiding global variables
- Define conventions and provide boilerplate code to support:
  - Project directory structure, including deployment config files
  - Bootstrapping (all the procedural stuff that initializes the environment and kicks off request handling)
  - Request routing (deciding what to do based on a web request)
  - Custom 'page actions'
    (objects that are invoked to _do something_ in response to a web request)
  - CRUD REST APIs (using [PHP CMIP-REST](https://github.com/EarthlingInteractive/PHPCMIPREST))
  - Templated views
  - Component classes
  - Logging
  - Error handling
  - Authentication and authorization
  - Common webapp features such as user logins
  - Deploying to shared hosting environments
  - Deploying using containers (e.g. Docker)[*](#configuration)
- Separate construction of HTTP responses from actually sending them
  using [Nife](https://github.com/ryprop/Nife) Blob and HTTP Reponse objects.


## Philosophy

Someone tried to make me use Laravel once but it didn't work because I hated it too much.
So I formalized some of the stuff I'd been doing things
and called it "Phrebar" because rebar is cool and this is like the PHP version.

The framework should be subservient to the application, not the other
way around.  With regard to which specific technologies are used, it
should provide good suggestions, but otherwise be as unbiased as
possible.

Template code is provided to help get new projects off the ground
quickly with the expectation that not all of its components will be
needed (and those not needed can be deleted), and that those parts
that are used will evolve over time to better suit the needs of the
application.  Code and design patterns that are found to be commonly
useful may be rolled into libraries or later versions of the template
project.

Application code calls into library code.  Callbacks into application
code from library code might sometimes be necessary but should not be
the norm.  A good example of this is principle in action is the
'router'.  Most MVC frameworks provide a router object that you
configure, which in turn calls your controllers to handle requests.
Routers in Phrebar-based apps are simply application-defined classes.

90% of REST services are doing simple CRUD operations, the behavior of
which can be entirely defined by:

1. Metadata about the schema (names, types of classes and fields), and
2. Authorization rules

Therefore functions that implement CRUD services should be centralized
while still allowing special cases without hackery.


## Request-Handling Process

### Layer 0: Procedural code

Is the procedural bootstrapping code that initializes autoloaders and
error handlers and otherwise puts the PHP interpreter into a state
where it's easier to work with, encapsulates the request and context,
calls layer 1 to handle the request, and spits responses back to the
browser.


### Layer 1: Request handler

At a conceptual level, the web service(s) can be thought of as a
single function-with-side effects that takes a web request and returns
a web response.  Responses are somewhat easier to encapsulate since we
control their construction, and Phrebar projects represent
them using Nife_HTTP_Response objects.
Requests are harder, since PHP scatters request data
throughout several global variables.  Session variables are tough to
model as they may be thought of as part of the request or as part of
the response.  To keep things organized, we'll treat them as 'context'
which is closely related to the request (the request is done within a
context), but separate from either the request or the response.

In pseudocode, our web service might be defined as:

  mutable WebService :: (Request, mutable Context) -> (Response)

Read: Request and response are immutable, context is mutable, and the
web service itself is mutable in that it may make modifications to
itself (e.g. altering database records) in the course of handling a
request.


### Layer 2: Request handling details

The process of handling a web request is divided into the following steps:

- Validate authentication, if any
  - return a 401 if invalid
  - if valid, $user = the authenticated user
  - if not given, $user = 'anonymous'
- Translate the request into an 'action' object ($action)
  - the action will include information about what to return
    as its response
- Ensure that $action is structurally valid ('makes sense')
  - returng a 409 if not
- Determine if $user is allowed to do $action
  - if not, return a 401 if $user = 'anonymous', 403 otherwise
- If the action is a side-effect-free (e.g. any GET/HEAD/OPTIONS request)
  and If-None-Match/If-Modified-Since (or similar) headers are given,
  check if the action would return an updated resource
  (how to do this will vary between actions and may require some coordination with the database)
  - if not, return a 304 (Not Modified)
- Invoke $action (passing a context object, which includes $user), returning its result
  - Further authorization and validation of the action may be done
    during (or after, in the case of search results) its invocation
  - If invoking the action throws an exception, that exception is turned
    into a response object and returned.


### Request -> Action parsing

For web pages and form submissions, there will be custom request
parsing code and custom actions for most cases.

API calls can be parsed automatically based on the schema for simple
CRUD actions.  Special-case CRUD actions may require custom parsing
code which falls back to the general parser for unrecognized cases
using something like the following:

```
class My_Cool_App_RequestParser {
  protected $generalRequestParser;
  
  public function __construct(callable $generalRequestParser) {
    $this->generalRequestParser = $generalRequestParser;
  }
  
  function parseRequest($request) {
    if( $this->isSomeSpecialCaseRequest($request) ) {
      return $this->someSpecialCaseRequestToAction($request);
    } else if( $this->isSomeOtherSpecialCaseRequest($request) ) {
      return $this->someOtherSpecialCaseRequestToAction($request);
    } else {
      return call_user_func($this->generalRequestParser, $request);
    }
  }
}
```


### Action invocation

The application defines a single object that is responsible for
'invoking' actions.  We'll call it $actionInvoker.  It will take an
action and a request-handling context as parameters, and return a web
response (or throw an exception).

Many actions will be one-off special cases where it makes sense to
define the behavior of the action as part of the action's class
definition.  These action classes will probably be identified by
implementing some marker interface and having an
```__invoke($context)```.  In this case, $actionInvoker will just
```call_user_func($action, $context)```.

Other actions, such as CRUD ones, won't define their own behavior
directly, but instead contain declarative information about what they
are trying to accomplish, such as 'Get user record #12, returning the
result as JSON, following the JSON-style naming convention'.  In such
cases, the component in charge of invoking actions will have to have
rules for figuring out how to invoke the action based on its data.


## Configuration

Phrebar was designed to make deploying to shared hosting environments,
but it should be just as easy to deploy the same codebase as
a Docker image.

For shared hosting, it's convenient to store deployment-specific
configuration in files alongside the application code.  This is what the ```config/```
directory is for.

For containerized applications it's nice to store configuration in
environment variables.  Newer versions of the framework support
that as an alternative or in combination with config files
via the [PHPConfigLoader](https://github.com/EarthlingInteractive/PHPConfigLoader)
library, which can be easily shimmed into old applications, too.

Application code can just call ```$component->getConfig("foo/bar")``` and not
be concerned with how that config variable was stored.


## Related projects

- [PHPProjectInitializer](http://github.com/EarthlingInteractive/PHPProjectInitializer)
  can be used to set up a new project following this pattern.
- [PHPTemplateProject](http://github.com/EarthlingInteractive/PHPTemplateProject)
  contains the template code used by PHPProjectInitializer.
- [PHPProjectUtils](http://github.com/EarthlingInteractive/PHPProjectUtils)
  contains utility scripts for building a project, including the running of database upgrade scripts.
- [PHPSchema](https://github.com/EarthlingInteractive/PHPSchema)
  defines classes for working with schema metadata.
- [CMIP-REST](https://github.com/EarthlingInteractive/PHPCMIPREST)
  implements REST API services based on a schema definition.
- [PHPStorage](https://github.com/EarthlingInteractive/PHPStorage)
  is the storage abstraction layer used by CMIP-REST, but is
  also made available for use directly from application PHP code.


## Stuff that needs documenting

- The registry object
- Database functions and abstractions
- Best practices for using other packages
- CMIP-REST (possibly just referring to documentation in its repository)
