---
layout: post
title: "Consuming REST APIs with (Cloud) ABAP"
date: 2021-01-03
excerpt: "Using Cloud ABAP, we will create an HTTP service, and use it to serve dynamically, server-side rendered HTML. I.e., we will be doing web development in ABAP. Our use of ABAP will be quite similar to, for example, using Flask in Python. Of course, however, we don’t actually have a framework to use, so we will have to do some things – like the template rendering – ourselves. This tutorial includes every step of the process, from creating the HTTP service, to using the web app. To do this, I will use the demo use-case of a movie database."
---

*My article was originally published on [SAP Blogs](https://blogs.sap.com/2021/01/23/web-development-with-cloud-abap/)*

<h1>Preface</h1>
My first article on SAP Blogs – <em><a href="https://blogs.sap.com/2020/10/27/consuming-rest-apis-with-cloud-abap/">Consuming REST APIs with (Cloud) ABAP</a></em> – was quite well received, so I figured this one might also be an interesting read for ABAP developers. I read <a href="https://pacroy.medium.com/developing-apis-in-abap-just-rest-not-odata-d91cf899f7d3">this excellent article</a> on developing plain REST APIs in ABAP a while ago. I was looking for something like this because I wanted to develop an automated black-box testing solution for a huge ABAP application. I.e., only care about input &amp; output – a solution that is obviously worse than extensive unit tests, but a good start nonetheless. I wanted to compare outputs stored in CSVs, so I preferred to do it in Python. And to allow the Python script to communicate with the SAP system (and send/receive data to/from the ABAP application), I exposed the ABAP application through a plain REST API, as described in the linked article by Chairat Onyaem (Par).

While doing this, I was struck with the idea that, given that this is possible, you have all necessary capabilities to use ABAP for web development. Sometime later, I tested this, and, indeed, I could make a simple web app using only ABAP. This weekend, I decided to do the same, but in the SAP Cloud Platform ABAP Environment, i.e., in Cloud ABAP. Instead of using ICF nodes, and the <em>cl_rest_http_handler</em> class – which is prohibited for use in Cloud ABAP – I achieved it using an HTTP service. The result was actually so simple, yet powerful, that I decided to share it in this blog post.

The way I see it, possible applications include developing a front-end for very small applications. Given that all that is necessary (from the front-end side) is knowledge of the typical web technologies – HTML/CSS/JS – this can be outsourced easily to any developer, even if they have never touched anything SAP-related. Another possibility is to use it for demos, where the purpose would be making something very quickly, just to test/show from the front-end side some back-end functionality. And, IMHO, it is a cool thing to see something like this is technically possible – even if it is never actually used in practice.
<h1>Introduction</h1>
Using Cloud ABAP, we will create an HTTP service, and use it to serve dynamically, server-side rendered HTML. I.e., we will be doing web development in ABAP. Our use of ABAP will be quite similar to, for example, using Flask in Python. Of course, however, we don’t actually have a framework to use, so we will have to do some things – like the template rendering – ourselves. This tutorial includes every step of the process, from creating the HTTP service, to using the web app. To do this, I will use the demo use-case of a movie database.

All of the code is available in <a href="https://github.com/s7oev/awd">this GitHub repository</a>. <strong>Keep in mind that there is some extra ABAP logic in there that is used for storing the web app’s HTML.</strong> I briefly explain what it is in the demo section, but you can pretty much ignore it. The only important classes are <a href="https://github.com/s7oev/awd/blob/main/src/zcl_ss_awd_demo.clas.abap">Demo (zcl_ss_awd_demo)</a>, which is the auto-generated handler class, <a href="https://github.com/s7oev/awd/blob/main/src/zcl_ss_awd_helper.clas.abap">Helper (zcl_ss_awd_helper)</a> and <a href="https://github.com/s7oev/awd/blob/main/src/zcl_ss_awd_movies.clas.abap">Movies (zcl_ss_awd_movies)</a>. All of them and their purposes will be explained below.

<em>Note about the naming: ss = Stoyko Stoev and awd = ABAP Web Development</em>
<h1>Creating the HTTP service</h1>
This step could not be any simpler – after logging in Eclipse ABAP Development Tools with our SAP Cloud Platform user data, we choose a package, select <em><u>New &gt; Other ABAP Repository Object &gt; HTTP Service</u></em>, input some name… and we are done! We then see the URL and the Handler Class for this service, which the system auto-generated for us. Besides this, a lot of other auto-generated things now show up in our package:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd0.png" alt="Auto-generated%20objects%20related%20to%20the%20HTTP%20service" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Auto-generated objects related to the HTTP service</p>
However, for the purpose of our tutorial, we don’t really care about anything other than the Handler Class and the URL.
<h1>Implementing the Handle Request method and app paths</h1>
Our newly generated Handler Class implements the <em>if_http_service_extension</em> interface. This means that we have to have the interface’s method <em>handle_request </em>in our class. I’ll present the different elements of its logic in the easiest to grasp order, while the full code will be available at the end of the section (as well as in the GitHub repository).

To get the path the user is requesting, we do the following:
<pre class="language-abap"><code>DATA(path) = request-&gt;get_header_field( '~path_info' ).</code></pre>
&nbsp;

So, for example, if the user is on
https://&lt;id&gt;.abap-web.eu10.hana.ondemand.com/sap/bc/http/sap/zss_awd_demo/browse, the method call will return the string “/browse”. We will be using this akin to Flask’s routes, if you are familiar with Flask (if not, doesn’t matter, it’s not important at all). Then, we format the path:
<pre class="language-abap"><code>path = zcl_ss_awd_helper=&gt;format_path( path ).</code></pre>
&nbsp;

I will not be showing the format_path method in this section (it is available in the GitHub), as all it does is: (1) remove the first character; and (2) convert the string to uppercase. So, “/browse” simply becomes “BROWSE”.

The next step is to dynamically call a method with this name:
<pre class="language-abap"><code>CALL METHOD me-&gt;(path).</code></pre>
&nbsp;

This is why we needed to convert to uppercase (dynamic method call only works with uppercase). From here on, when we want to add a new route to our application, we simply add a new method with the route name that we want… and that’s literally all of it! So, if we want to actually implement a “/browse” route, we add a new method called “browse”. This is (visually) a bit similar to doing this in Flask – which is what I meant we will be using the path akin to Flask’s routes:
<pre class="language-python"><code>@app.route('/browse')
def browse():</code></pre>
&nbsp;

But, again, if you don’t know Python and/or Flask, this is not important at all.

Then, what we would like to do is to be able to use the request and response objects inside the methods for the different paths. We obviously need this, to be able to return something to the user, and to also know what exactly they are requesting. To do this, we will simply make the request and response instance attributes:
<pre class="language-abap"><code>    me-&gt;request = request.
    me-&gt;response = response.</code></pre>
&nbsp;

Now, for the impatient, let’s immediately see the fruits of our labor. With only this, we can simply add a method <em>hello_world</em> (without any parameters), and add the following code inside:
<pre class="language-abap"><code>  METHOD hello_world.
    response-&gt;set_text( 'Hello, world!' ).
  ENDMETHOD.</code></pre>
The method <em>set_text</em> of the response object <em>“sets the HTTP body of this entity to the given string data”</em>.

So, if we go to https://&lt;our HTTP Service URL&gt;/hello_world, we see this greeting indeed!
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd0.5.png" alt="Hello%2C%20world%21" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Hello, world!</p>
Here’s how the full code looks at this point in time:
<pre class="language-abap"><code>CLASS zcl_ss_awd_demo DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES:
      if_http_service_extension.

  PRIVATE SECTION.
    DATA:
      request  TYPE REF TO if_web_http_request,
      response TYPE REF TO if_web_http_response.

    METHODS:
      hello_world.
ENDCLASS.



CLASS zcl_ss_awd_demo IMPLEMENTATION.
  METHOD if_http_service_extension~handle_request.
    me-&gt;request = request.
    me-&gt;response = response.

    DATA(path) = request-&gt;get_header_field( '~path_info' ).
    path = zcl_ss_awd_helper=&gt;format_path( path ).

    CALL METHOD me-&gt;(path).
  ENDMETHOD.


  METHOD hello_world.
    response-&gt;set_text( 'Hello, world!' ).
  ENDMETHOD.
ENDCLASS.</code></pre>
&nbsp;

To help you get acquainted also with the request object, let’s add some dynamicity to our application. Let’s, instead of saying <em>Hello, world!</em>, say <em>Hello, &lt;name&gt;!</em>, where &lt;<em>name&gt;</em> will be a URL query parameter. The way to get a query parameter is to use the method <em>get_form_field</em> of the request object. We modify our <em>hello_world</em> method as follows:
<pre class="language-abap"><code>  METHOD hello_world.
    DATA(name) = request-&gt;get_form_field( 'name' ).
    response-&gt;set_text( |Hello, { name }!| ).
  ENDMETHOD.</code></pre>
&nbsp;

Now, if we request the URL https://&lt;our HTTP Service URL&gt;/hello_world?name=&lt;some name&gt; we will see a dynamic greeting! Here’s the demo:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd1.png" alt="Hello%2C%20Bobby%21" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Hello, Bobby!</p>
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd2.png" alt="Hello%2C%20Dessy%21" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Hello, Dessy!</p>
Anyway, back to our <em>handle_request</em> method. We are almost done, but let’s add some edge-case considerations. One, if the path is empty, serve the homepage, and two – if there is no such method in the class, serve a 404 page.
<pre class="language-abap"><code>  METHOD if_http_service_extension~handle_request.
    me-&gt;request = request.
    me-&gt;response = response.

    DATA(path) = request-&gt;get_header_field( '~path_info' ).

    IF strlen( path ) EQ 0.
      home(  ).
      RETURN.
    ENDIF.

    path = zcl_ss_awd_helper=&gt;format_path( path ).
    TRY.
        CALL METHOD me-&gt;(path).
      CATCH cx_sy_dyn_call_illegal_method.
        not_found_404(  ).
    ENDTRY.
  ENDMETHOD.</code></pre>
&nbsp;

With this, we conclude the method! These less than 15 lines of code (without the white-space) are all it takes to create a basic web app in ABAP – impressive! But how do we serve HTML? Very simple! We just pass HTML as the string parameter for <em>set_text</em> of the response object, like this for example:
<pre class="language-abap"><code>  METHOD home.
    response-&gt;set_text( |&lt;html&gt; &lt;head&gt; &lt;title&gt; Home &lt;/title&gt; &lt;/head&gt; &lt;body&gt; &lt;p&gt;Home page&lt;/p&gt; &lt;/body&gt; &lt;/html&gt;| ).
  ENDMETHOD.</code></pre>
&nbsp;

We do something similar with our 404 method:
<pre class="language-abap"><code>  METHOD not_found_404.
    response-&gt;set_text( |&lt;html&gt; &lt;head&gt; &lt;title&gt; 404 &lt;/title&gt; &lt;/head&gt; &lt;body&gt; &lt;p&gt;404: no such page exists&lt;/p&gt; &lt;/body&gt; &lt;/html&gt;| ).
  ENDMETHOD.</code></pre>
And we have our working web app!
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd3.png" alt="Home%20page" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Home page</p>
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd4.png" alt="404%20page" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">404 page</p>
Admittedly, that’s not very impressive, is it? So let’s do something more interesting!
<h1>Demo use-case</h1>
Here, I’ll have to make one note. Due to ABAP’s restriction of not more than 255 characters per line, I decided to store the HTML in a DB table. I also exposed the table using a REST service supporting all the HTTP verbs, so that I can edit the HTML files locally and then store them in the DB by connecting with a Python script to the REST service. But that’s not really related to our main goal, so I will not be explaining how I did this. Still, the code for this is also in the GitHub if you would like to check it out.

To get the relevant HTML, I added a method to my helper class that fetches it from the DB table:
<pre class="language-abap"><code>  METHOD get_page_html.
    SELECT SINGLE content
      FROM zss_awd_html
      WHERE page_name = @page_name
      INTO @result.
  ENDMETHOD.</code></pre>
&nbsp;

<em>(in a productive environment, you probably wouldn’t want to have SQL code in your Helper class, but we are aiming for simplicity)</em>

So, I created the HTML files <em>index.html</em>, <em>browse.html</em>, <em>add.html</em>, <em>404.html</em> that are all using <a href="https://www.w3schools.com/howto/tryhow_make_a_website.htm">this W3 Schools template</a>. The only interesting thing happening inside is that I use variables that would later be replaced with the dynamically rendered content. Once again, similar to Flask (through the jinja rendering) – and once again, totally cool if you are unfamiliar with Flask. To identify the variables, I prefix them with <em>$</em>. Here’s one example:
<pre class="language-markup"><code>&lt;div class="row"&gt;
  &lt;div class="main"&gt;
    &lt;h2&gt;Here is our movie catalog:&lt;/h2&gt;
    $movies
  &lt;/div&gt;
&lt;/div&gt;</code></pre>
Here, $movies would be replaced with a list of all the movies in the database.

Speaking of the movies database, here’s how the Movies (zss_awd_movies) table looks like:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd5.png" alt="Movies%20Table" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Movies Table</p>
The underscore at the end of the year field is because <em>year</em> is a reserved word and cannot be used as a fieldname.

I also created a class to read from and write to the table, that also has one method that converts an ABAP internal table to an HTML unordered list:
<pre class="language-abap"><code>CLASS zcl_ss_awd_movies DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    TYPES:
      movies_tt TYPE TABLE OF zss_awd_movies WITH EMPTY KEY.

    CLASS-METHODS:
      create
        IMPORTING name          TYPE string
                  year          TYPE i
        RETURNING VALUE(result) TYPE sy-subrc,

      read_all
        RETURNING VALUE(result) TYPE movies_tt,

      itab_to_html_tab
        IMPORTING movies        TYPE zcl_ss_awd_movies=&gt;movies_tt
        RETURNING VALUE(result) TYPE string.
ENDCLASS.



CLASS zcl_ss_awd_movies IMPLEMENTATION.
  METHOD create.
    DATA(movie) = VALUE zss_awd_movies( name = name year_ = year ).

    INSERT zss_awd_movies FROM @movie.

    result = sy-subrc.
  ENDMETHOD.


  METHOD read_all.
    SELECT *
      FROM zss_awd_movies
      INTO TABLE @result.
  ENDMETHOD.


  METHOD itab_to_html_tab.
    result = |&lt;ul&gt;|.

    LOOP AT movies REFERENCE INTO DATA(movie).
      result &amp;&amp;= |&lt;li&gt;{ movie-&gt;name } ({ movie-&gt;year_ })&lt;/li&gt;|.
    ENDLOOP.

    result &amp;&amp;= |&lt;/ul&gt;|.
  ENDMETHOD.
ENDCLASS.</code></pre>
<em>(again, in production you probably should not have one class with both DB operations and business logic – but we are going for simplicity)</em>

Now, let’s start to implement some web app-related logic in our handler class! First, let’s get the relevant HTMLs:
<pre class="language-abap"><code>  METHOD home.
    response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'index' ) ).
  ENDMETHOD.


  METHOD browse.
    response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'browse' ) ).
  ENDMETHOD.


  METHOD add.
    response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'add' ) ).
  ENDMETHOD.


  METHOD not_found_404.
    response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( '404' ) ).
  ENDMETHOD.</code></pre>
&nbsp;

With this, we have a static, but at least good looking web app! Here’s what everything up till now looks like:

[embed]https://youtu.be/DyM1XxCCcIA[/embed]

Towards the end of the video, we see the “/add” route, which looks as follows in HTML code:
<pre class="language-markup"><code>&lt;div class="row"&gt;
  &lt;div class="main"&gt;
    &lt;h2&gt;Extend our movie catalog:&lt;/h2&gt;
    &lt;form action="/sap/bc/http/sap/zss_awd_demo/add" method="get"&gt;
	  &lt;input type="hidden" id="action" name="execute" value="X"&gt;
      &lt;label for="name"&gt;Movie name:&lt;/label&gt;&lt;br&gt;
      &lt;input type="text" id="name" name="name"&gt;&lt;br&gt;&lt;br&gt;
      &lt;label for="year"&gt;Movie year:&lt;/label&gt;&lt;br&gt;
      &lt;input type="text" id="year" name="year"&gt;&lt;br&gt;&lt;br&gt;
	  &lt;input type="submit" value="Create"&gt;
	&lt;/form&gt;
  &lt;/div&gt;
&lt;/div&gt;</code></pre>
Now, we will implement our first logic handling in a route! We add a new HTML template, <em>executed_add.html</em> which will show up in the “/add” route after the form is submitted. Here’s what the server-side logic looks like:
<pre class="language-abap"><code>  METHOD add.
    IF request-&gt;get_form_field( 'execute' ) = abap_true.
      response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'executed_add' ) ).
    ELSE.
      response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'add' ) ).
    ENDIF.
  ENDMETHOD.</code></pre>
This works because of the hidden input called <em>execute</em> which has as value ‘X’ ( = abap_true). The <em>executed_add.html </em>template has two variables – response_title and response_body:
<pre class="language-markup"><code>&lt;div class="row"&gt;
  &lt;div class="main"&gt;
    &lt;h2&gt;$response_title&lt;/h2&gt;
    $response_body
  &lt;/div&gt;
&lt;/div&gt;</code></pre>
And from now on, when we press the <em>Create </em>button, we are still in the “/add” route, but, under the hood, we get served the <em>executed_add.html</em> template:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd6.png" alt="executed_add.html" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">executed_add.html</p>
Of course, we are still not doing any rendering at this point. So let’s get there! To do this, we will use our <em>Helper </em>class. First, we introduce a public type in the class:
<pre class="language-abap"><code>    TYPES:
      BEGIN OF var_and_content_s,
        variable TYPE string,
        content  TYPE string,
      END OF var_and_content_s,

      var_and_content_tt TYPE TABLE OF var_and_content_s WITH EMPTY KEY.</code></pre>
This would be used by passing the variable name (for example, <em>response_title</em>) and with what content it should be replaced (for example, <em>“Successfully added movie!”</em>). The actual rendering will happen in a method, also in the Helper, called <em>render_html</em> with the following signature:
<pre class="language-abap"><code>      render_html
        IMPORTING html                TYPE string
                  var_and_content_tab TYPE var_and_content_tt
        RETURNING VALUE(result)       TYPE string</code></pre>
And the following implementation:
<pre class="language-abap"><code>  METHOD render_html.
    result = html.

    LOOP AT var_and_content_tab REFERENCE INTO DATA(var_and_content).
      result = replace( val = result sub = |${ var_and_content-&gt;variable }| with = var_and_content-&gt;content ).
    ENDLOOP.
  ENDMETHOD.</code></pre>
Let’s test it in our “/add” route:
<pre class="language-abap"><code>  METHOD add.
    IF request-&gt;get_form_field( 'execute' ) = abap_true.
      DATA(html) = zcl_ss_awd_helper=&gt;get_page_html( 'executed_add' ).

      html = zcl_ss_awd_helper=&gt;render_html( html = html var_and_content_tab = VALUE #(
        ( variable = 'response_title' content = 'Testing rendering... (response_title)' )
        ( variable = 'response_body' content = 'Testing rendering... (response_content)' )  ) ).

      response-&gt;set_text( html ).
    ELSE.
      response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'add' ) ).
    ENDIF.
  ENDMETHOD.</code></pre>
This time, after pressing the create button, we see the rendered HTML!
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd7.png" alt="Testing%20rendering%20%28successfully%21%29" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Testing rendering (successfully!)</p>
Of course, at this point, we are not really creating anything. But we are almost there! We already have the class (<em>Movies</em>) to handle this, with its <em>create </em>method. All we have to do is just call it, evaluate the subrc it returned, and set the response based on this! There’s two options: first, all’s good – the movie was created (subrc 0) and second, the movie already exists (subrc 4). Here’s what we do:
<pre class="language-abap"><code>  METHOD add.
    IF request-&gt;get_form_field( 'execute' ) = abap_true.
      " add the movie
      DATA(subrc) = zcl_ss_awd_movies=&gt;create( name = request-&gt;get_form_field( 'name' )
        year = CONV #( request-&gt;get_form_field( 'year' ) ) ).

      " prepare response based on subrc
      DATA(response_title) = SWITCH #( subrc WHEN 0 THEN |Successfully added movie { request-&gt;get_form_field( 'name' ) }|
        ELSE |Sorry, we could not add the movie...| ).

      DATA(response_content) = SWITCH #( subrc WHEN 0 THEN |Thank you for extending our movie database!|
        ELSE |...because we already have it in our database!| ).

      " return the dynamically rendered HTML response
      DATA(html) = zcl_ss_awd_helper=&gt;get_page_html( 'executed_add' ).

      html = zcl_ss_awd_helper=&gt;render_html( html = html var_and_content_tab = VALUE #(
        ( variable = 'response_title' content = response_title )
        ( variable = 'response_body'  content = response_content ) ) ).

      response-&gt;set_text( html ).
    ELSE.
      response-&gt;set_text( zcl_ss_awd_helper=&gt;get_page_html( 'add' ) ).
    ENDIF.
  ENDMETHOD.</code></pre>
Now, let’s test it! We can add Titanic:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd8.png" alt="Adding%20Titanic" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Adding Titanic</p>
After doing this, we get a success message:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd9.png" alt="Successfully%20added%20Titanic" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Successfully added Titanic!</p>
But did we actually succeed? Let’s use the SQL console to find out…
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd10.png" alt="Output%20for%20selecting%20everything%20from%20the%20Movies%20table" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Output for selecting everything from the Movies table</p>
It definitely seems so! In fact, if we now try to add Titanic again we will not succeed:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd11.png" alt="We%20already%20have%20Titanic%21" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">We already have Titanic!</p>
This means we are almost finished with our application! Let’s just add the functionality to list the movies in the “/browse” route:
<pre class="language-abap"><code>  METHOD browse.
    DATA(movies) = zcl_ss_awd_movies=&gt;read_all(  ).

    DATA(html) = zcl_ss_awd_helper=&gt;render_html(
     html = zcl_ss_awd_helper=&gt;get_page_html( 'browse' ) var_and_content_tab = VALUE #(
     ( variable = 'movies' content = zcl_ss_awd_movies=&gt;itab_to_html_tab( movies ) ) ) ).

    response-&gt;set_text( html ).
  ENDMETHOD.</code></pre>
Which results in the following:
<p style="overflow: hidden;margin-bottom: 0px"><img class="aligncenter" src="https://blogs.sap.com/wp-content/uploads/2021/01/awd12.png" alt="Our%20movie%20catalog" /></p>
<p class="image_caption" style="text-align:center;font-style:italic;, Arial, sans-serif">Our movie catalog</p>
To reward ourselves for making it till here, here’s a video of this, albeit simple, fully fledged web app:

[embed]https://youtu.be/bflMbQ1z7aI[/embed]
<h1>Limitations</h1>
One important limitation I found, and is not discussed above, is that if the HTML form is sending the data with a POST method, we get a 403 error (forbidden). This is why I went with the implementation using a GET to the same route, and an <em>execute </em>query parameter. I haven’t really investigated much why this happens. A prima vista, I think this is probably caused by the lack of CSRF token, which is required for executing modifying HTTP verbs (such as POST or DELETE). It might be possible to overcome this with a JavaScript modifying the sent request. It might also not be. The issue itself might have a completely different cause, instead of a CSRF token.
<h1>Conclusion</h1>
This tutorial shows that, from a technical perspective, it is completely possible to use ABAP for developing simple web applications. There might be some possible applications of this – in fact, if you do end up using, or have already used, ABAP in this way – please let me know in the comments! It is also important to mention that all of this is completely possible to replicate in an on-premise system, by using ICF nodes and the <em>cl_rest_http_handler </em>class.

I am hoping this has been an interesting read!
