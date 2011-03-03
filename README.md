# rewrite
`rewrite` is an implementation of [MarkLogic's URL Rewriter for HTTP Application Servers][11]. `rewrite` aims to provide an expressive language that allows you to specify REST applications. This is intended to make your routing logic simple and easy to maintain:

      <routes>
        <root> dashboard#show </root> 
        <resource name="inbox"> <!-- no users named inbox --> 
          <member action="sent"/> 
        </resource> 
        <resource name=":user"> 
          <constraints>  
            <user type="string" match="^[a-z]([a-z]|[0-9]|_|-)*$"/> 
          </constraints> 
          <member action="followers"/> <!-- no repo named followers --> 
          <resource name=":repo"> 
            <constraints>  
              <repo match="^[a-z]([a-z]|[0-9]|_|-|\.)*$"/> 
            </constraints> 
            <member action="commit/:commit"> 
              <constraints>  
                <commit type="string" match="[a-zA-Z0-9]+"/> 
              </constraints> 
            </member> 
            <member action="tree/:tag" /> 
            <member action="forks" /> 
            <member action="pulls" /> 
            <member action="graphs/impact" /> 
            <member action="graphs/language" /> 
          </resource> 
        </resource>
      </routes>

Routes are [matched in the order you specified][17] and they can be [nested][18]. `rewrite` also enables you to hide specific routes from users given specific constraints.

`rewrite` is designed to work with [MarkLogic][2] Server only. However it can easily be ported to another product that understands XQuery and has similar capabilities. `rewrite` is heavily inspired in the [Rails 3.0 routing][4].

## Usage

In your HTTP Application Server configuration make `rewrite.xqy` the default rewriter script.

*This section doesn't cover how to set up an HTTP Application Server in MarkLogic. If you are a beginner I suggest you start by browsing the [MarkLogic Developer Community site][7] or sign up for some [training][8].*

Place the `lib` folder of `rewrite` in your application `root`. Still in the `root`  create a new file named `rewrite.xqy` with the following contents:

     xquery version "1.0-ml" ;
     
     import module namespace r = "routes.xqy" at "/lib/routes.xqy" ;
     
     declare variable $routesCfg := 
       <routes>
         <root> users#list </root>
         <get path="users/:id">
           <to> users#show </to>
         </get>
       </routes> ;
     
     r:selectedRoute( $routesCfg )

Now with the `rewrite` in place:

* Request `/` will be dispatched to `/resource/users.xqy?action=list`
* Request `/users/dscape`  will be dispatched to `/resource/users.xqy?action=show&id=dscape`

You can [customize the file path][19] and / or  [store configurations in a file][20]. If you are curious on how the translation from path to file is done refer to "Supported Features". 

Here's an example of how your `users.xqy` might look like:

     xquery version "1.0-ml";
     
     import module namespace u = "user.xqy" at "/lib/user.xqy";
     import module namespace h = "helper.xqy" at "/lib/helper.xqy";
     
     declare function local:list() { u:list() };
     declare function local:get()  { u:get( h:id() ) } ;
     
     try          { xdmp:apply( h:function() ) } 
     catch ( $e ) { h:error( $e ) }

A centralized [error handler][14] can also be used removing the need for a `try catch` statement. Refer to the wiki section on [using an error handler][21] for instructions.

*This assumes a hypothetical `users.xqy` XQuery library that actually does the work of listing users and retrieving information about a user. It also contains a `helper.xqy` module. The `helper.xqy` module is contained in lib as an example but is not part of `rewrite`, so you can/should modify it to fit your needs; or even create your fully fledged [MVC][10] framework.*

## Sample Application

Not yet. Include redirect-to because it can't be proven without an extra file. Include errors.xqy as well!

## Supported Functionality

In this section we describe the DSL that you can use to map your routes
to where they should be dispatched. This is meant to give you overall understanding of the functionality without having to read the code.

### 1. Routes

###  ✔ 1.1. root 
     Request       : GET /
     routes.xml    : <routes> <root> server#version </root> </routes> 
     Dispatches to : /resource/server.xqy?action=ping

###  ✔ 1.2. verbs
Some properties that are common amongst all verbs: get, put, post, delete, and head. For information regarding these properties refer the wiki section on [How Verbs Work][22] 

### 1.2.1.1. dynamic paths
If you think about a website like twitter.com the first level path is given to users, e.g. `twitter.com/dscape`. This means that we need to route all our first levels display user information.

`rewrite` exposes that functionality with dynamic paths. For the twitter example we would have something like:

     Request       : GET /dscape
     routes.xml    : <routes> 
                       <get path=":user">
                         <to> user#get </to>
                       </get>
                     </routes>
     Dispatches to : /resource/user.xqy?action=get&user=dscape

The colon in `:user` lets the routing algorithm know that `:user` shouldn't be evaluated as a string but rather as a dynamic resource. You can even combine dynamic resources with static paths:

     Request       : GET /user/dscape
     routes.xml    : <routes> 
                       <get path="user/:id">
                         <to> user#get </to>
                       </get>
                     </routes>
     Dispatches to : /resource/user.xqy?action=get&id=dscape

###  1.2.1.2. bound parameters
There are two symbols that are special `:resource` maps to the name of a controller in your application, and `:action` maps to the name of an action within that controller. When you supply both in a route it will be evaluated by calling that action on the specified resource. Everything else will be passed as field values and can be retrieve by invoking the [xdmp:get-request-field][12] function.

The following example is a route with bound parameters that will match `/users/get/1` to resource `users`, action `get`, id `1`:

     Request       : GET /users/get/1
     routes.xml    : <routes> 
                       <get path=":resource/:action/:id"/>
                     </routes>
     Dispatches to : /resource/users.xqy?action=get&id=1

###  1.2.1.3 redirect-to
You can specify a redirect by using the `redirect-to` element inside your route:

     Request       : GET /google
     routes.xml    : <routes> 
                       <get path="google">
                         <redirect-to> http://www.google.com </redirect-to> 
                       </get>
                     </routes>
     paths.xml     : <paths>
                       <resourceDirectory>/</resourceDirectory>
                       <redirect>dispatcher</redirect>
                     </paths>
     Dispatches to : /dispatcher.xqy?url=http%3a//www.google.com

The `rewriter` script cannot process re-directs natively in MarkLogic Server 4.2, as the output of any rewrite script must be a xs:string.

The current implementation of `rewrite` sends all redirects to a `redirect` dispatcher with an url-encoded option `url` that contains the url. 

The dispatcher can have any logic you like. Here is an example of a possible `redirect.xqy` dispatcher:

     let $url := xdmp:get-request-field( "url" )
     return if ( $url )
            then xdmp:redirect-response( xdmp:url-decode( $url ) )
            else fn:error()

If you are using `redirect-to` don't forget to place a `redirect.xqy` in the resource directory.

An alternative is to user a error handler as described in usage.

###  ✔ 1.2.2. get 
     Request       : GET /list
     routes.xml    : <routes> 
                       <get path="list"> <to> article#list </to> </get>
                     </routes>
     Dispatches to : /resource/article.xqy?action=list

###  ✔ 1.2.3. put 
     Request       : PUT /upload
     routes.xml    : <routes>
                       <put path="upload"> <to> file#upload </to> </put>
                     </routes>
     Dispatches to : /resource/file.xqy?action=upload

###  ✔ 1.2.4. post
     Request       : POST /upload
     routes.xml    : <routes>
                       <post path="upload"> <to> file#upload </to> </post>
                     </routes>
     Dispatches to : /resource/file.xqy?action=upload

###  ✔ 1.2.5. delete 
     Request       : DELETE /all-dbs
     routes.xml    : <routes>
                       <delete path="all-dbs"> 
                         <to> database#delete-all </to>
                       </delete>
                     </routes>
     Dispatches to : /resource/database.xqy?action=delete-all

###  ✔ 1.2.6. head 
     Request       : HEAD /
     routes.xml    : <routes> <head> <to> server#ping </to> </head> </routes>
     Dispatches to : /resource/server.xqy?action=ping

###  ✔ 1.3. resources
It's often the case when you want to perform all CRUD (Create, Read, Update, Delete) actions on a single resource, e.g. you want to create, read, update and delete users. RESTful architectures normally map those actions to HTTP verbs such as GET, PUT, POST and DELETE.

When you create a resource in `rewrite` you expose these actions:

<table>
  <tr>
    <th>Verb</th>
    <th>Path</th>
    <th>Action</th>
    <th>Used in</th>
    <th>Notes</th>
  </tr>
  <tr>
    <td>GET</td>
    <td>/users</td>
    <td>index</td>
    <td>Web-Services, Web-Applications</td>
    <td>Displays a list of all users</td>
  </tr>
  <tr>
    <td>GET</td>
    <td>/users/:id</td>
    <td>get</td>
    <td>Web-Services, Web-Applications</td>
    <td>Display information about a specific user</td>
  </tr>
  <tr>
    <td>PUT</td>
    <td>/users/:id</td>
    <td>put</td>
    <td>Web-Services, Web-Applications</td>
    <td>Creates or updates a user</td>
  </tr>
  <tr>
    <td>DELETE</td>
    <td>/users/:id</td>
    <td>delete</td>
    <td>Web-Services, Web-Applications</td>
    <td>Deletes a user</td>
  </tr>
  <tr>
    <td>POST</td>
    <td>/users</td>
    <td>post</td>
    <td>Web-Applications</td>
    <td>No special meaning</td>
  </tr>
  <tr>
    <td>GET</td>
    <td>/users/new</td>
    <td>new</td>
    <td>Web-Applications</td>
    <td>Return a form for creating a user.</td>
  </tr>
  <tr>
    <td>GET</td>
    <td>/users/:id/edit</td>
    <td>edit</td>
    <td>Web-Applications</td>
    <td>Return a form for editing the user.</td>
  </tr>
</table>

By default post, new and edit actions are created. If you are creating a web-service and have no interest in them you can change this behavior by simply passing a `webservice="true"` attribute to the resources specification.

The following example explains a single match against one of the multiple routes a resource creates. Please explore further examples (or try it out yourself) if you want to have a better understanding of resources.

     Request       : PUT /users/1
     routes.xml    : <routes> 
                       <resources name="users" webservice="true"/> 
                     </routes>
     Dispatches to : /resource/users.xqy?action=put&id=1

### 1.3.1. includes
Resources are really great cause they save you all the trouble of writing all those routes all the time (especially when order matters and you have to make sure you get it right).

### 1.3.1.1. member
Sometimes you will need to include one or more actions that are not part of the default resource, e.g. you might want to create a enable or disable one of your users. 

For this you need the resource to respond to `PUT /users/dscape/enabled` and understand that should re-enable the user. This action runs against a specific user - that's why we call it `member` include. Here's an example of how you can express that in `rewrite`:

     Request       : PUT /users/dscape/enabled
     routes.xml    : <routes> 
                       <resources name="users">
                         <member action="enabled" for="PUT,DELETE"/>
                       </resources>
                     </routes>
     Dispatches to : /resource/users.xqy?action=enabled&id=dscape

If you are curious about the DELETE - it's simply there to allow you to disable a user the RESTful way. If you don't pass the `for` attribute then  GET will be the default.

###  1.3.1.2. set
Another type of action you might need to add are global actions, e.g. searching all users in full text. 

We call this a set include and express it as follows:

     Request       : PUT /users/search?q=foo
     routes.xml    : <routes> 
                       <resources name="users">
                         <member action="enabled" for="PUT,DELETE"/>
                         <set action="search"/>
                       </resources>
                     </routes>
     Dispatches to : /resource/users.xqy?action=search&q=foo

Member and set includes are not exclusive of each other and you can use as many as you want in your resources.

###  ✔ 1.4. resource
Some resource only expose a single item, e.g. your about page. While you might want to be able to perform CRUD actions on a about page there is only one about page so using `resources` (plural) would be useless.

When you create a `resource` in `rewrite` you expose these actions:

<table>
  <tr>
    <th>Verb</th>
    <th>Path</th>
    <th>Action</th>
    <th>Used in</th>
    <th>Notes</th>
  </tr>
  <tr>
    <td>GET</td>
    <td>/about</td>
    <td>get</td>
    <td>Web-Services, Web-Applications</td>
    <td>Display about section</td>
  </tr>
  <tr>
    <td>PUT</td>
    <td>/about</td>
    <td>put</td>
    <td>Web-Services, Web-Applications</td>
    <td>Creates or updates the about section</td>
  </tr>
  <tr>
    <td>DELETE</td>
    <td>/about</td>
    <td>delete</td>
    <td>Web-Services, Web-Applications</td>
    <td>Deletes the about section</td>
  </tr>
  <tr>
    <td>POST</td>
    <td>/about</td>
    <td>post</td>
    <td>Web-Applications</td>
    <td>No special meaning</td>
  </tr>
  <tr>
    <td>GET</td>
    <td>/about/edit</td>
    <td>edit</td>
    <td>Web-Applications</td>
    <td>Return a form to create/edit the about section</td>
  </tr>
</table>

By default post, and edit actions are created. If you are creating a web-service and have no interest in them you can change this behavior by simply passing a `webservice="true"` attribute to the resources specification.

The following example illustrates a resource:

     Request       : GET /about
     routes.xml    : <routes> 
                       <resource name="about"/> 
                     </routes>
     Dispatches to : /resource/about.xqy?action=get

### 1.4.1. dynamic resource
As with `get`, `put`, etc, you can also create a dynamic resource by prefixing the name with `:`. Here's an example of using this to create a database:

     Request       : PUT /documents
     routes.xml    : <routes> 
                       <resource name=":database"/> 
                     </routes>
     Dispatches to : /resource/database.xqy?action=put&database=Documents

### 1.4.2. member

      Request       : PUT /car/ignition
      routes.xml    : <routes> 
                        <resources name="car">
                          <include action="ignition" for="PUT,DELETE"/>
                        </resources>
                      </routes>
      Dispatches to : /resource/users.xqy?action=ignition

### 2. Extras

###  ✔ 2.1. mixed paths
     Request       : GET /user/43
     routes.xml    : <routes> 
                       <get path="user/:id">
                         <to> user#show </to>
                       </get>
                     </routes>
     Dispatches to : /resource/user.xqy?action=show&id=43

###  ✔ 2.2. static
If no match is found `rewrite` will dispatch your query to a /static/ folder where you should keep all your static files. This way you don't have to create routing rules for static files.

     Request       : GET /css/style.css
     routes.xml    : <routes> <root> server#version </root> </routes> 
     Dispatches to : /static/css/style.css

###  ✔ 2.3. constraints

### 2.3.1 bound parameters
When you bound parameters you sometime need to validate that they are valid. For our twitter example we would want to validate that `dscape` is indeed a proper `:user` using a [regular expression][13]. In a simpler case you might want to check that an `:id` is a decimal number. You can do that using the [XML schema datatypes][16]

     Request       : GET /user/dscape
     routes.xml    : <routes> 
                       <get path="user/:id">
                         <constraints>
                           <id type="integer"/>
                         </constraints>
                         <to> user#show </to>
                       </get>
                     </routes>
     Dispatches to : /static/user/dscape  (no match of type xs:integer, trying static)

Regular Expression Example:

     Request       : GET /lost-username/bill@sample.com
     routes.xml    : <routes> 
                       <get path="lost-username/:email">
                         <constraints>
                           <email 
                             type="string"  match="[A-Za-z0-9._%-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,4}"/>
                         </constraints>
                         <to> users#recoverUserName </to>
                       </get>
                     </routes>
     Dispatches to : /resource/users.xqy?action=recoverUserName?email=bill@sample.com

### 2.3.2 privileges
Privilege constraints make routes visible to the user if he is part of a role with the specified permission:

     Request       : GET /list
     routes.xml    : <routes> 
                       <privileges>
                         <execute> 
                           http://marklogic.com/xdmp/privileges/admin-ui
                         </execute>
                         <uri>
                           http://marklogic.com/xdmp/triggers/
                         </uri>
                       </privileges>
                       <get path="list"> <to> article#list </to> </get>
                     </routes>
     Dispatches to : /resource/article.xqy?action=list

Many applications use the same login do all accesses to the database. Hence it might be useful to explicitly pass the username in the privilege constraints. This is how you can express this in `rewrite`:

     Request       : GET /list
     routes.xml    : <routes> 
                       <privileges for="user">
                         <execute> 
                           http://marklogic.com/xdmp/privileges/xdmp-eval
                         </execute>
                       </privileges>
                       <get path="list"> <to> article#list </to> </get>
                     </routes>
     Dispatches to : /resource/article.xqy?action=list

While very flexible this also means your `routes.xml` is no longer static. You will have to pass the current user every time a request comes.

### 2.3.3. lambdas
The most flexible way of ensuring constraints is to run an XQuery lambda function. An example usage for a lambda in a constraint would be "only show the user information that pertains to the currently logged-in user"

     Request       : GET /user/admin
     routes.xml    : <routes> 
                       <get path="user/:id">
                         <lambda>
                           xdmp:get-current-user() = $id
                         </lamda>
                         <to> user#get </to>
                       </get>
                     </routes>
     Dispatches to : /resource/user.xqy?action=get&id=admin

The bound parameters will be available in the lambda as an xs:string external variable; e.g. `:id` will be available as `$id`. 

###  ✔ 2.4. scopes
Scopes allow you to reuse your constraints for multiple routes:

     Request       : GET /user/admin
     routes.xml    : <routes>
                       <scope>
                         <constraints>
                           <id type="string"/>
                         </constraints>
                         <lambda>
                           xdmp:get-current-user() = $id
                         </lamda>
                         <put path="user/:id">
                           <to> user#put </to>
                         </put>
                         <get path="user/:id">
                           <to> user#get </to>
                         </get>
                       </scope>
                     </routes>
     Dispatches to : /resource/user.xqy?action=get&id=admin


###  ✕ mvc goodies
Content Negotiation and other MVC goodies are deliberately not bundled in `rewrite`. 

The objective of `rewrite` is to simplify the mapping between external URLs and internal file paths. If you are curious about MVC and other topics you can look at some of my on-going work at the [dxc][5] project.

For your convenience some functions that might be necessary to run an application with `rewrite` have be placed in the `/lib/helper.xqy` library.

## Contribute

Everyone is welcome to contribute. 

1. Fork rewrite in github
2. Create a new branch - `git checkout -b my-branch`
3. Test your changes
4. Commit your changes
5. Push to your branch - `git push origin my-branch`
6. Create an pull request

The documentation is severely lacking. Feel free to contribute to the wiki if 
you think something could be improved.

### Running the tests

To run the tests simply point an MarkLogic HTTP AppServer to the root of `rewrite`

You can run the tests by accessing:
(assuming 127.0.0.1 is the host and 8090 is the port)

    http://127.0.0.1:8090/test/

Make sure all the tests pass before sending in a pull request!

### Report a bug

If you want to contribute with a test case please file a [issue][1] and attach 
the following information:

* Request Method
* Request Path
* routes.xml
* Request Headers (if relevant)
* paths.xml (if relevant)

This will help us be faster fixing the problem.

This is not the actual test that we run (you can see a list of those in test/index.html) but it's all the information we need from a bug report.

## Roadmap

If you are interested in any of these (or other) feature and don't want to wait just read the instructions on "Contribute" and send in your code. I'm also very inclined to implement these features myself so it might be that a simple email is enough to motivation for me to get it done.

v0.1

* Nested Resources
* Improve read-me, re-factor all notes to wiki
* Sample app

v0.2

* Translated Paths for resources
* Generating Paths and URLs from code
* Route Globbing
* Namespaces, e.g. /admin/user/1/edit
* Make singular resources map to plural controllers
* Restricting Resource(s) Routes
* Make redirect-to flexible
* Allows bound constraints containing / in the values (test exists)

### Known Limitations

In this section we have the know limitations:

* Bound parameters containing / are not supported.
* Nested Sections don't propagate constraints between levels.

### Dynamic paths

When using dynamic paths it is impractical to keep separate files for each user you have. So in the `/:user` example you map them to `user.xqy` and pass the username as a parameter, e.g. `user.xqy?user=dscape`. 

Please keep this in mind when developing your web applications. Other request fields can have the same name and your users can even inject fields, e.g. `user.xqy?user=dscape&user=hkstirman`. This framework will always give you the dynamic path as the first parameter so a safe way of avoiding people tampering with your logic is to get the first field only:

     xdmp:get-request-field( 'user' ) [1]

In the `user.xqy?user=dscape&user=hkstirman` this would return:

     dscape

On previous versions of `rewrite` dynamic paths where prefixed by `_`, so `user` would be `_user`. I choose to make it explicit so people stumble upon it faster and realize they still need to carefully protect themselves against  hacks like this.

## Meta

* Code: `git clone git://github.com/dscape/rewrite.git`
* Home: <http://github.com/dscape/rewrite>
* Discussion: <http://convore.com/marklogic>
* Bugs: <http://github.com/dscape/rewrite/issues>

(oO)--',- in [caos][3]

[1]: http://github.com/dscape/rewrite/issues
[2]: http://marklogic.com
[3]: http://caos.di.uminho.pt
[4]: http://edgeguides.rubyonrails.org/routing.html
[5]: http://github.com/dscape/dxc
[6]: https://github.com/dscape/dxc/blob/master/http/http.xqy#L27
[7]: http://developer.marklogic.com
[8]: http://www.marklogic.com/services/training.html
[9]: http://xqzone.marklogic.com/pubs/4.2/apidocs/Ext-7.html#xdmp:document-get
[10]: http://en.wikipedia.org/wiki/Model–View–Controller
[11]: http://docs.marklogic.com/4.2doc/docapp.xqy#display.xqy?fname=http://pubs/4.2doc/xml/dev_guide/appserver-control.xml%2313050
[12]: http://developer.marklogic.com/pubs/4.2/apidocs/AppServerBuiltins.html#xdmp:get-request-field
[13]: http://en.wikipedia.org/wiki/Regular_expression
[14]: http://docs.marklogic.com/4.2doc/docapp.xqy#display.xqy?fname=http://pubs/4.2doc/xml/dev_guide/appserver-control.xml
[15]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
[16]: http://www.w3.org/TR/xmlschema-2
[17]: https://github.com/dscape/rewrite/wiki/Routes-are-ordered
[18]: https://github.com/dscape/rewrite/wiki/Nested-Routes
[19]: https://github.com/dscape/rewrite/wiki/Customize-File-Path
[20]: https://github.com/dscape/rewrite/wiki/Loading-Configuration-from-Files
[21]: https://github.com/dscape/rewrite/wiki/Using-an-Error-Handler
[22]: https://github.com/dscape/rewrite/wiki/How-Verbs-Work