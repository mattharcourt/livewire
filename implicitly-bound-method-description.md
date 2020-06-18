## Synopsis (TLDR)

There have been a number of issues/requests related to, and a few solutions proposed for,
dependency injection (#92, #127, PR#128, #380, #1116)
and implicit model binding (#295, #701, #796, PR#934)
in component mount and action methods.


This PR proposes a solution to address dependency injection and implicit model binding
for component mount and action methods by extending the binding features provided by
Laravel's container, and encapsulating in one place, the logic currently implemented
separately for component mount and component actions.


**Jump to:**
  [Results](#Results)
| [Background](#Background)
| [Implementation](#Implementation)
| [Additional Considerations](#additional-considerations)


## Results

- Dependency injection is handled by Laravel's container logic.
- Implicit model binding is handled in conjunction with dependency injection
(both types of parameter substitution are encapsulated in one place).
- Component mount and component actions use a common implementation for
dependency injection and implicit model binding.


## Background

Much of this is likely already known, but the information is provided here for completeness.


### Dependency Injection

Laravel's container method provides the desired dependency injection through the
[`Illuminate\Container\BoundMethod`](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Container/BoundMethod.php)
class used by `app()->call(...)`.
In addition to providing dependency injection when calling Closures, array callbacks
(e.g. `[$instance, $method]`), and class method strings (e.g. `'{$class}@{$method}'`)
`BoundMethod` also applies custom method binding to override the container's
default dependency injections for a particular method.
*[the method binding and other features may not be a Livewire requirement,
but come for free with this solution]*.


### Implicit Model Binding

Most are familiar with implicit model binding through Laravel's route model binding,
which implicitly binds a model for controller methods based on a parameter
supplied in the request (as defined in when the route is registered).
*Note: this PR does *not* address ***route***-model binding for components (there
are other issues/PRs addressing route model binding - e.g. #115, #1068).*
This PR addresses implicit model binding where a route is not involved.


Since the pattern of invoking a component action method is similar to invoking
a controller method (both being triggered by some sort of request from the client),
it is only natural to desire similar binding behavior in component methods.
As proposed in PR#934, this PR leverages the interface implemented for route
model binding in the `Model` to resolve the implicit binding for component
methods, but does so within the context of Laravel's
[`\Illuminate\Container\BoundMethod`](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Container/BoundMethod.php)
logic for resolving dependencies.

This enables automatic injection of a model instance when an appropriate
model key is provided as a parameter. For example:

When calling a **component action** with a model key, as in
```javascript
    <button wire:click="showPost(post_id)">Show</button>
```
the appropriate model instance will be injected when the action method expects a model
```php
    public function showPost(Post $post)
```
And a **component mount** method will receive the appropriate model
instance when it expects a model, as in
```php
    public function mount(Post $post)
```
and the component is initialized in a Blade template with a model key, as in
```blade
    <livewire:show-post, ['post' => $post_id]>
```
(*This use case for implicit model binding when initializing a component
in a Blade template is probably less frequent as the model itself will
typically be available,
but this implementation of dependency injection for mount also makes
implicit model binding available*).



###  Parameter Name Binding

When calling a Livewire component action from the client, the method parameters
are simply listed in order in the function call and are therfore not packaged
as `key => value` pairs with the parameter names assigned as keys.
So the order of the parameter values implies the binding to the parameters.

In order to accomplish implicit model binding, the parameter values need
to be correctly matched with each parameter.
But dependency injection means that the absolute position cannot be relied
on when matching parameters with their values
(i.e. `value[0]` may not be for `parameter[0]` if `parameter[0]` represents an
injected dependency).

A straightforward way to handle this is to bind each parameter with its
value by name -- i.e. re-key the parameter values using
the appropriate parameter name as the key.
Then when it comes time to do implicit model binding, it's simple to identify
the correct parameter value to be used.

And as identified in #910 and elsewhere, parameter name binding is also required
in order to utilize
[`\Illuminate\Container\BoundMethod`](https://github.com/laravel/framework/blob/7.x/src/Illuminate/Container/BoundMethod.php)
for dependency injection.
(not doing so will disrupt dependency injection and cause default values
to override passed values).


## Implementation

This PR extends Laravel container binding capabilities by defining
```php
class ImplicitlyBoundMethod extends \Illuminate\Container\BoundMethod
```
and overriding the
[`getMethodDependencies`](https://github.com/laravel/framework/blob/b87233f48da0b4f219adebd851acd22058dfd551/src/Illuminate/Container/BoundMethod.php#L117)
function of `BoundMethod`
so that it handles parameter name binding and implicit model binding
in addition to dependency injection.
In the parent class,
the `getMethodDependencies` function loops through the parameters
defined for the method, and passes each reflected method parameter
to a function that locates the parameter value - injecting dependencies
or retrieving default values when necessary.

The
[`ImplicitlyBoundMethod`]()
override of this loop inserts two
additional function calls:

1. [`substituteNameBindingForCallParameter`]()
to ensure that the appropriate parameter value is associated with the parameter
by name (using the parameter name as a key in the parameter value array).
    - Candidates for dependency injection are left for handling
    by the parent class.
    - If the reflected parameter doesn't already have a parameter
    value associated with it by name, the next parameter value keyed
    by integer is bound to it by name.
    - When a variadic parameter is encountered, all remaining values
    are reindexed so they are keyed from zero in the array assigned to
    the variadic parameter during the actual method call.

2. [`substituteImplicitBindingForCallParameter`]()
to handle any implicit model binding that is expected.
    - Implicit bindings are attempted on any parameters defined as a class
    that implements the `Illuminate\Contracts\Routing\UrlRoutable` interface,
    and whose parameter value is not already an instance of the expected class.
    - Parameters expecting other classes are assumed to be candidates for
    dependency injection and are left to be handled by the parent class.

> <sub>A note about the code -
> conventions established in the parent class are followed.
> Thus the static functions, pass-by-reference, and
> `$parameter` vs `$parameters`, where
> `$parameter` refers to the reflected method parameter and
> `$parameters` refers to the array of parameter values.


## Additional Considerations
- Just as how the container applies dependency injection (in `app()->call()`),
implicit model binding can be applied to any methods, functions or closures
(when called using `ImplicitlyBoundMethod`).
- As in route model binding, implicit model binding takes precedence over
dependency injection (if a parameter is defined as a class that can be
implicitly bindable, implicit model binding will be performed even if
the container could inject an instance as a dependency).
- Route model binding for component mount remains separate (it is
fundamentally different since it must parse the request for parameter values).
- Other considerations or opportunities for improvement?
