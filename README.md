JSON API helpers for Laravel 5
=====

[![Build Status](https://travis-ci.org/EarthlingInteractive/laravel-jsonapi.svg?branch=master)](https://travis-ci.org/EarthlingInteractive/laravel-jsonapi)

Make it a breeze to create a [jsonapi.org](http://jsonapi.org/) compliant API with Laravel 5. 

Code forked from echo-it/laravel-jsonapi project by Ronni Egeriis Persson.

Installation
-----

1. Add `earthling-interactive/laravel-jsonapi` to your composer.json dependency list

3. Run `composer update`.

### Requirements

* PHP 5.4+
* Laravel 5


Using laravel-jsonapi
-----

This library is made with the concept of exposing models in mind, as found in the RESTful API approach.

In few steps you can expose your models:

1. **Create a route to direct the requests**

    In this example, we use a generic route for all models and HTTP methods:

    ```php
    Route::any('{model}/{id?}', 'ApiController@handleRequest');
    ```

2. **Create your controller to handle the request**

    Your controller is responsible to handling input, instantiating a handler class and returning the response.

    ```php
		 <?php namespace App\Http\Controllers;

        use EarthlingInteractive\JsonApi\Request as ApiRequest;
        use EarthlingInteractive\JsonApi\ErrorResponse as ApiErrorResponse;
        use EarthlingInteractive\JsonApi\Exception as ApiException;
        use Request;

        class ApiController extends Controller
        {
            public function handleRequest($modelName, $id = null)
            {
                /**
                 * Create handler name from model name
                 * @var string
                 */
                $handlerClass = 'App\\Handlers\\' . ucfirst($modelName) . 'Handler';
        
                if (class_exists($handlerClass)) {
                    $url = Request::url();
                    $method = Request::method();
                    $include = ($i = Request::input('include')) ? explode(',', $i) : $i;
                    $sort = ($i = Request::input('sort')) ? explode(',', $i) : $i;
                    $filter = ($i = Request::except('sort', 'include', 'page')) ? $i : [];
                    $content = Request::getContent();
                    
                    $page = Request::input('page');
                    $pageSize = null;
                    $pageNumber = null;
                    if($page) {
                        if(is_array($page) && !empty($page['size']) && !empty($page['number'])) {
                            $pageSize = $page['size'];
                            $pageNumber = $page['number'];
                        } else {
                             return new ApiErrorResponse(400, 400, 'Expected page[size] and page[number]');
                        }
                    }
                    $request = new ApiRequest(Request::url(), $method, $id, $content, $include, $sort, $filter, $pageNumber, $pageSize);
                    $handler = new $handlerClass($request);
        
                    // A handler can throw EarthlingInteractive\JsonApi\Exception which must be gracefully handled to give proper response
                    try {
                        $res = $handler->fulfillRequest();
                    } catch (ApiException $e) {
                        return $e->response();
                    }
                    
                    return $res->toJsonResponse();
                }
        
                // If a handler class does not exist for requested model, it is not considered to be exposed in the API
                return new ApiErrorResponse(404, 404, 'Entity not found');
            }
        }
    ```

3. **Create a handler for your model**

    A handler is responsible for exposing a single model.

    In this example we have create a handler which supports the following requests:

    * GET /users (ie. handleGet function)
    * GET /users/[id] (ie. handleGet function)
    * PUT /users/[id] (ie. handlePut function)
    
    Requests are automatically routed to appropriate handle functions.

    ```php
      <?php namespace App\Handlers;
      
      use Symfony\Component\HttpFoundation\Response;
      use App\Models\User;
      
      use EarthlingInteractive\JsonApi\Exception as ApiException;
      use EarthlingInteractive\JsonApi\Request as ApiRequest;
      use EarthlingInteractive\JsonApi\Handler as ApiHandler;
      use Request;
      
      /**
       * Handles API requests for Users.
       */
      class UsersHandler extends ApiHandler
      {
          const ERROR_SCOPE = 1024;
          
          /*
          * List of relations that can be included in response.
          * (eg. 'friend' could be included with ?include=friend)
          */
          protected static $exposedRelations = [];
          
          /**
           * Handles GET requests. 
           * @param EarthlingInteractive\JsonApi\Request $request
           * @return EarthlingInteractive\JsonApi\Model|Illuminate\Support\Collection|EarthlingInteractive\JsonApi\Response|Illuminate\Pagination\LengthAwarePaginator
           */
          public function handleGet(ApiRequest $request)
          {
              //you can use the default GET functionality, or override with your own 
              return $this->handleGetDefault($request, new User);
          }
          
          /**
           * Handles PUT requests. 
           * @param EarthlingInteractive\JsonApi\Request $request
           * @return EarthlingInteractive\JsonApi\Model|Illuminate\Support\Collection|EarthlingInteractive\JsonApi\Response
           */
          public function handlePut(ApiRequest $request)
          {
              //you can use the default PUT functionality, or override with your own
              return $this->handlePutDefault($request, new User);
          }
      }
    ```

    > **Note:** Extend your models from `EarthlingInteractive\JsonApi\Model` rather than `Eloquent` to get the proper response for linked resources.

Current features
-----

According to [jsonapi.org](http://jsonapi.org):

* [Resource Representations](http://jsonapi.org/format/#document-structure-resource-representations) as resource objects
* [Resource Relationships](http://jsonapi.org/format/#document-structure-resource-relationships)
   * Only through [Inclusion of Linked Resources](http://jsonapi.org/format/#fetching-includes)
* [Compound Documents](http://jsonapi.org/format/#document-structure-compound-documents)
* [Sorting](http://jsonapi.org/format/#fetching-sorting)
* [Filtering](http://jsonapi.org/format/#fetching-filtering)
* [Pagination] (http://jsonapi.org/format/#fetching-pagination)

The features in the Handler class are each in their own function (eg. handlePaginationRequest, handleSortRequest, etc.), so you can easily override them with your own behaviour if desired. 
	

Wishlist
-----

* [Relationship URLs](http://jsonapi.org/format/#document-structure-resource-relationships)  e.g. /users/[id]/links/friend
* [Resource URLs](http://jsonapi.org/format/#document-structure-resource-urls)
* [Updating Relationships](http://jsonapi.org/format/#crud-updating-relationships)
* [Sparse Fieldsets](http://jsonapi.org/format/#fetching-sparse-fieldsets)
* Strict checking of application/vnd.api+json in content-type and Accept Headers
