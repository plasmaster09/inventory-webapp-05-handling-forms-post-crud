# Part 05: Implementing CRUD operations: Using Forms and POST requests

This tutorial follows after:
[Part 04: Connecting the Layers: Rendering templates with data](https://github.com/atcs-wang/inventory-webapp-04-connecting-layers-templates)

In this tutorial, we will continue to work across all 3 layers (Browser, Web Server, Database) to implement standard "CRUD" data operations for the inventory items. In the process, we will learn to use HTML forms and HTTP POST requests to upload user-entered data to the server.

Our app currently allows users to see (Read) data from the database, but not make any changes to the database; we will add functionality for the other three CRUD operations that cause database changes: Create, Update, and Delete.

## (5.0) The big picture for CRUD operations

CRUD is an acronym for the 4 basic data storage operations: Create, Read, Update, and Delete. SQL queries have 4 commands that map onto these operations (INSERT, SELECT, UPDATE, DELETE).

Data-based web-applications typically provide interactive ways for users to perform various CRUD operations for the various kinds of data from the browser.

> In most cases, of course, not *every* user should be allowed to do *every* kind of CRUD operation on *every* kind of data. We will explore ways to differentiate between different user *roles* and how to authorize specific operations later.

Recall from last tutorial that our web-app's "flow" generally follows this pattern: 

>```
> Browser --- HTTP request ---> App Server
>                               App Server --- SQL query --> Database
>                               App Server <-- results ---- Database
> Browser <-- HTTP response --- App Server
>```

1. **The web server receives an HTTP request from a browser**
2. **The web server makes a query to the database**.
3. **The web server waits for results of the query**
4. **The web server uses the query results to form and send the HTTP response back to the browser**

The exact details of how each of the 4 steps works can vary, but every operation we implement in this tutorial follows this pattern.

By the end of this tutorial, we will have implemented 5 different CRUD operations that operate on the `stuff` table. Each follows the above pattern, but the details vary at each step. 

Here's a table that summarizes each operation:

| Operation        | HTTP Request + URL     | SQL Command | HTTP response              |
|------------------|------------------------|-------------|----------------------------|
| Read all items   | GET  /stuff            | SELECT      | render stuff.ejs           |
| Read one item    | GET  /stuff/:id        | SELECT      | render item.ejs            |
| Delete item      | GET  /stuff/:id/delete | DELETE      | redirect to GET /stuff     |
| Create item  | POST /stuff            | INSERT      | redirect to GET /stuff/:id |
| Update item      | POST /stuff/:id        | UPDATE      | redirect to GET /stuff/:id |

First, let's review the two Read operations we've implemented in the last tutorial. Then, let's implement the other 3 operations (Delete, Create, and Update).

## (5.1) Review: The Read operations

In the last tutorial, we already implemented the 2 Read operations: one for a specific item, and one for the entire inventory.

Its worth reviewing how both fit the 4-step pattern. Here's the code in `app.js` that defines the two Read operation routes:

```js
// define a route for the stuff inventory page
const read_stuff_all_sql = `
    SELECT 
        id, item, quantity
    FROM
        stuff
`
app.get( "/stuff", ( req, res ) => {
    db.execute(read_stuff_all_sql, (error, results) => {
        if (error)
            res.status(500).send(error); //Internal Server Error
        else {
            res.render('stuff', { inventory : results });
        }
    });
} );

// define a route for the item detail page
const read_stuff_item_sql = `
    SELECT 
        id, item, quantity, description 
    FROM
        stuff
    WHERE
        id = ?
`
app.get( "/stuff/item/:id", ( req, res ) => {
    db.execute(read_stuff_item_sql, [req.params.id], (error, results) => {
        if (error)
            res.status(500).send(error); //Internal Server Error
        else if (results.length == 0)
            res.status(404).send(`No item found with id = "${req.params.id}"` ); // NOT FOUND
        else {
            let data = results[0]; // results is still an array
            // data's object structure: 
            //  { id: ____, item: ___ , quantity:___ , description: ____ }
            res.render('item', data);
        }
    });
});
```

Here's the 4 steps again, but broken down specifically for the Read operations:

>```
> Browser --- request: GET URL ------> App Server
>                                      App Server --- SELECT query--> Database
>                                      App Server <-- results ARRAY ---- Database
> Browser <- response: RENDERED PAGE-> App Server
>```

1. **The web server receives a GET HTTP request from a browser** - either by entering the URL into their browser, or clicking a hyperlink.
2. **The web server makes a SELECT query to the database**. If the URL is `/stuff`, we SELECT for all data in the `stuff` table. If the URL is `/stuff/:id`, we only SELECT for the one row in `stuff` with the matching `id`.
3. **The web server waits for results of the query**; if successful, the results are an *array* of objects representing a set of row entries.
4. **The web server uses the query results to RENDER a page and sends it in the HTTP response back to the browser**; in both cases, HTML pages are rendered from an EJS template (either `stuff.ejs` or `item.ejs`) using the query results. The browser receives the HTML and displays it.

As we implement the other CRUD operations, you will see how they work similarly and differently from the Read operation flow. 

## (5.2) Implementing the Delete Operation

Although it is last in the CRUD acronym, Delete is most similar to the flow of the Read operation. So let's start there:

| Operation        | HTTP Request + URL     | SQL Command | HTTP response              |
|------------------|------------------------|-------------|----------------------------|
| Delete item      | GET  /stuff/:id/delete | DELETE      | redirect to GET /stuff     |


Similar to how our app interprets to GET requests to the URL `/stuff/item/:id` as a Read operation for the entry with given `id`, our app will interpret GET requests to the URL `stuff/item/:id/delete` as a Delete operation for the entry with given `id`.

### (5.2.1) Writing the DELETE item SQL query:

To delete an entry from the `stuff` table, matching a given `id`, we can write a generalized query like this:

```sql
DELETE 
FROM
    stuff
WHERE
    id = ?
```
> This is recorded in `db/queries/crud/delete_stuff.sql`. Although we will not directly use this file in this tutorial, we'll refer to it later on.

This operation doesn't really return any interesting data, although it can provide confirmation of how many rows get deleted. (Since the `id` is a primary key, it should be just 1 or 0).

### (5.2.2) Handling a Delete item GET request:

Add the following code to `app.js`, below the existing routes:

```js
// define a route for item DELETE
const delete_item_sql = `
    DELETE 
    FROM
        stuff
    WHERE
        id = ?
`
app.get("/stuff/item/:id/delete", ( req, res ) => {
    db.execute(delete_item_sql, [req.params.id], (error, results) => {
        if (error)
            res.status(500).send(error); //Internal Server Error
        else {
            res.redirect("/stuff");
        }
    });
})
```

Let's breakdown what the new code does; 
>```
> Browser --- request: GET URL ------> App Server
>                                      App Server --- SELECT query ---> Database
>                                      App Server <-- confirmation ---- Database
> Browser <- response: REDIRECT------> App Server
>```

1. **The web server receives a GET HTTP request from a browser** - either by entering a URL into their browser, or clicking a hyperlink.
2. **The web server makes a DELETE query to the database**, using the `id` in the URL to fill in the ? in the query. (You may recall this is known as a "prepared statement".)
3. **The web server waits for results of the query**; the contents of the results returned are not particularly important, but we want to know when the request is complete and that there was no error.
4. **The web server uses the query results to send a "redirect" HTTP response back to the browser**; rather than rendering a page, the server response is a *redirect* (HTTP status code 302) - essentially an instruction for the browser to send *another* GET request to a different URL. In this case, a redirect to the `/stuff` URL navigates the user back to the inventory page, where the user can visually confirm the removal of the item.  
    > Alternatively, the server could respond by rendering a special page with a message confirming the success of the deletion, possibly with a link to `/stuff`. Such a confirmation page might be more clearly communicate a successful Delete operation, but a redirect is simpler and cuts out what might be a tedious extra click for the user.

#### (5.2.2.1) Testing a Delete item GET request:

At this point, we can test out the route handler by entering URLs into your browser of the format `/stuff/item/:id/delete`, replacing `:id` with the `id` values of entries in the database's `stuff` table (e.g. 1, 2, etc.). Each time, you should be redirected back to the `/stuff` inventory page, where the deleted item should no longer be visible.

>The server logs should report a status code of 302 (REDIRECT) for `GET /stuff/item/:id/delete`, followed by a `GET /stuff`.

Once you've tried it out, you can re-run your `db_init.js` script to re-initialize your database afterwards:

```
> node db/db_init.js
```

### (5.2.3) Making the "Delete" button send a GET request

We don't really intend for users to type raw URLs into their browser to trigger a Delete; rather, we can use hyperlinked buttons.

In both the inventory page (template `stuff.ejs`) and the item detail page (template `item.ejs`), there are currently nonfunctional "Delete" buttons. 

Recall that, in the previous tutorial, we set the `href` attribute of each of the "Edit/Info" buttons (in `stuff.ejs`) to a unique URL, composed from the `id` of the corresponding item/row (`inventory[i]`), thus linking specific item detail page for that item: 

```ejs
<tbody>
    <% for (let i = 0; i < inventory.length; i++) { %>
        <tr>
            ...
            <td>
                <a class="btn-small waves-effect waves-light" href="/stuff/item/<%= inventory[i].id %>" >
                    <i class="material-icons right">edit</i>
                    Info/Edit
                </a>
                <a class="btn-small waves-effect waves-light red">
                    <i class="material-icons right">delete</i>
                    Delete
                </a>
            </td>
        </tr>
    <% } %>
</tbody>
```

As a result, each "Edit/Info" button on each table row was associated with a unique URL. Clicking the button sends a GET request to the app server, "following the link".

In a very similar manner, we can also give the "Delete" buttons in `stuff.ejs` unique URLs based on the same `id`: 

```ejs
    <a class="btn-small waves-effect waves-light red" href="/stuff/item/<%= inventory[i].id %>/delete">
        <i class="material-icons right">delete</i>
        Delete
    </a>
```

Similarly, the "Delete" button on the item detail page (template `item.ejs`) needs to similarl link to a URL which matches the `id` of the data being rendered on that page: 

```ejs
    <a class="btn-large waves-effect waves-light red right"  href="/stuff/item/<%= id %>/delete">
        <i class="material-icons right">delete</i>Delete
    </a>
```

#### (5.2.3.1) Testing a Delete item GET request:


Confirm that clicking these updated "Delete" buttons on both pages triggers GET requests for URLs of the pattern `/stuff/item/:id/delete`, where `:id` matches the `id` of the data, and that you are redirected back to the `/stuff` page, where you can visually confirm the removal of the corresponding item.

>Again, the server logs should report a status code of 302 (REDIRECT) for `GET /stuff/item/:id/delete`, followed by a `GET /stuff`.

>*Side Note:* Using a hyperlink to produce a GET request is a functional way to implement deletion, but is a little strange conceptually. GET requests are intended for *accessing* data, not requesting *changes* to data. After all, we usually thinking clicking a blue hyperlinked URL will show us a page, not delete one. At the very least, a visual Button (rather than a blue hyperlink) doesn't feel too weird to cause deletion - but generally the tying of change operations to GET requests (which you can send as links or type into a URL bar) are still odd. 
> There are more natural (but more complex) ways to implement Delete operations, but we'll explore these much later.


## (5.3) What are POST requests?

So far, all of the interaction between the user and the web server have been through GET requests sent by the browser. These are typically triggered by

1. A URL being entered into the browser's URL bar
2. A hyperlink being clicked
3. A link to a resource on an HTML page (e.g. CSS, images, etc.) 
4. Being redirected to a URL by the server

In addition to HTTP GET requests, which are typically associated with *downloading* data, browsers can also send a different kind of HTTP request known as a POST requests, which are typically are used for *uploading* data. 
Like GET requests, POST requests are sent to a URL, but *also* include a **body**, containing the data to be uploaded to the server.

### (5.3.1) Making HTML forms send POST requests

Most commonly, POST requests are triggered via **HTML forms**, where users enter values into various inputs, then click a button that will "Submit" the form. The values enetered into the form inputs will be included in the body of the POST request.

To make an HTML `<form>` element that sends a POST request upon Submit, its HTML attribute `method` should be set to `"post"`. 

Here is a simple (hypothetical) example of an HTML form designed to send a POST request:

```html
<form method="post" action="/sample/url">
    <input type="text" name="name">
    <input type="number" name="age">
    <button type="submit">
</form>
```
> Don't put this code  anywhere; its just a hypothetical example.

Note the following details about standard form usage, as visible in this example:

- Optionally, the attribute `action` can be set to the URL to which the post requests should be sent.  If `action` is not defined, the form POSTs to the same URL of the current page.

- It is important that each `<input>` element in the form includes a `name` attribute, which describes the meaning of the input.  

- A `<button>` element with attribute `type="submit"` is usually included in a form too; clicking this button triggers the POST request.

Fortunately, our prototypes already have two forms mostly set up; the only thing that needs to be changed is their `method` must be set to "post". Then, we just need to set up POST routes (which work similarly to GET routes) on the server to handle the requests. See the sections below for step by step instructions.

>*Side notes*: If the `method` attribute of a `<form>` is instead explictly set to `"get"`, OR simply left undefined (like our prototype's forms currently are), HTML forms will send GET requests instead. The form will construct a special URL from the input values, acting like a customizable hyperlink. 
> If you click submit on our app's forms now, they appear to refresh the page. But if you look at the URL bar, the new URL includes a `?` symbol, followed by the various input names and values. For example, if you submit the `/stuff` page form with values "`thing`" and "`1`", you'd be redirected to the URL `/stuff?name=thing&quantity=1` 
> Express typically ignores the part of the URL after the `?` when resolving the matching GET route path, but those values can be retrieved as **URL query parameters** via the property `req.query`. **Search bars** are a common application of this kind of form and parameterized GET request; we will add one later!
> Form-created POST requests use a similar "URL-encoding" scheme to encode the inputs into the request body. However, POST requests do not update the URL bar of the browser. This is by design, as you wouldn't want someone to bookmark or share a link for a POST request.

### (5.3.2) Configuring Express to parse POST request bodies

In order for Express to handle POST requests sent from forms more easily, we need to add some built-in middleware. (no `npm install` needed!)

Add this line to the middleware section of `app.js` (around the other `app.use` lines):
```js
// Configure Express to parse URL-encoded POST request bodies (traditional forms)
app.use( express.urlencoded({ extended: false }) );
```

With this middleware installed, we can now use a (hypothetical) POST route definition like this:
```js
app.post("/sample/url", ( req, res ) => {
    //access to form values via req.body's properties:
    // For example:
    // req.body.name
    // req.body.age
    ...
}
```
> Again, don't put this code above anywhere; its just a hypothetical example.

The middleware parses any incoming POST requests, and attaches a property called `body` to the Request object `req`. `body` has sub-properties for each `input` in the form, sharing the same `name` and containing the `value` of the input. For example, if a form contains `<input name=entry>`, the post handler can access the user-entered value via `req.body.entry`.


## (5.4) Implementing the Create item operation 

Now that we know a bit about POST requests, and have configured Express middleware to parse POST request bodies, let's implement the Create operation for new items.

Here's the summary of what we'll do:

| Operation        | HTTP Request + URL     | SQL Command | HTTP response              |
|------------------|------------------------|-------------|----------------------------|
| Create item  | POST /stuff            | INSERT      | redirect to GET /stuff/:id |

### (5.4.1) Writing the INSERT query:
Let's start by writing the relevant SQL query for the Create operation.

We can use an INSERT statement to add a new entry to the `stuff` table, given initial values for the `item` and `quantity` columns. (`description` can be NULL for now)

No `id` is specified; the database will automatically generate a unique `id` for the new element.

```sql
INSERT INTO stuff
    (item, quantity)
VALUES
    (?, ?)
```
> A record of this query is in `db/queries/crud/insert_stuff.sql`.
> Also, we wrote a similar INSERT query in part 3 of the tutorials (`db/queries/init/insert_stuff_table.sql`), and used them in our database initializing script (`db/db_init.js`).

If successful, this operation returns the `id` (primary key) of the newly inserted entry.

### (5.4.2) Sending a Create item POST request: - the "Add Stuff" form

Now, let's make the "Add Stuff" form on the `/stuff' page send POST requests!

Update the `<form>` in the `stuff.ejs` template; set attributes `method="post"`, and (optionally) `action="/stuff"`, like so:

```html
    <form method="post" action="/stuff"> 
```
> Again, it is  not necessary to define `action` explicitly; by default, the action is the URL of the current page.

Note the `name` attribute of the various input elements; they have values "name" and "quantity":
```html
    <input type="text" name="name" id="nameInput" class="validate" data-length="32" required>
    ...
    <input type="number" name="quantity" id="quantityInput" value=1 required>
```

### (5.4.3) Handling a Create POST request:

Finally, let's write a route to receive the form's POST request and run the INSERT query. 

Add this code to your `app.js`, below the other routes.

```js
// define a route for item Create
const create_item_sql = `
    INSERT INTO stuff
        (item, quantity)
    VALUES
        (?, ?)
`
app.post("/stuff", ( req, res ) => {
    db.execute(create_item_sql, [req.body.name, req.body.quantity], (error, results) => {
        if (error)
            res.status(500).send(error); //Internal Server Error
        else {
            //results.insertId has the primary key (id) of the newly inserted element.
            res.redirect(`/stuff/item/${results.insertId}`);
        }
    });
})
```

Let's break down how the new route code we've just added implements the Create operation flow:

>```
> Browser --- request: POST URL -----> App Server
>                                      App Server --- INSERT query ---> Database
>                                      App Server <--- new row id ----- Database
> Browser <- response: REDIRECT------> App Server
>```

1. **The web server receives a POST HTTP request from a browser** - the user submits an HTML form. This POST handler receives the form data, which includes the values of the "name" and "quantity" inputs. These are accesible via `req.body.name` and `req.body.quantity`.
2. **The web server makes an INSERT query to the database**. Those input values are used to prepare and excute an `INSERT` SQL statement, which adds a new row to the `stuff` table with the form values. 
3. **The web server waits for results of the query**; upon successful execution of the SQL, the `results` returned from the database includes `insertId`, the value of the primary key for the newly inserted row.
4. **The web server uses the query results to form and send the HTTP response back to the browser**. Once again, rather than generate a new page the server sends a "redirect" to the item detail page of the newly created item.  The `results.insertId` is used to construct the matching URL.
    > Again, the server could respond by rendering a special page with a message confirming the success of the create operation, but it makes sense to simply show the user the new page that the operation has produced.

#### (5.4.3.1) Testing the Create item operation:

To test, restart your server, navigate to `/stuff`, fill out the form inputs, and click Submit. You should be redirected to a new item detail page, containing the data you entered. Return to the `/stuff` page and you should see the item there as well.

> You should notice in the server logs a `POST /stuff` request, with a status code of `302` for redirect, followed by a `GET /stuff/:id` request.

> You can also see the POST request/response from your browser's Developer Tools; in the Network tab, watch for a request to `stuff` as you submit the Form. Click on it to view the details.
>
>The "Headers" sub-tab should show:
>>Request URL: http://localhost:8080/stuff
>>Request Method: POST
>>Status Code: 302 Found
>
>And the "Payload" sub-tab should show:
>>name: ____
>>quantity: ____
>
>displaying the values entered into the form.


## (5.5) Implementing the Update item operation - 

The Update operation is extremely similar to the Create operation. Here's the summary of what we'll do:

| Operation        | HTTP Request + URL     | SQL Command | HTTP response              |
|------------------|------------------------|-------------|----------------------------|
| Update item      | POST /stuff/:id        | UPDATE      | redirect to GET /stuff/:id |

### (5.5.1)  Writing the UPDATE query:

Again, let's first write the relevant SQL: an UPDATE command which changes the `item`, `quantity`, and `description` values for an entry from the `stuff` table, matching a given `id`:

```sql
UPDATE
    stuff
SET
    item = ?,
    quantity = ?,
    description = ?
WHERE
    id = ?
```
> A record of this query is in `db/queries/crud/update_stuff.sql`.

This operation doesn't really return data, although it does provide confirmation of how many rows get updated. (Since the `id` is a primary key, it should be just 1 or 0)

###  (5.5.2) Sending an Update item POST request: - the "Edit" form

The "Edit" `<form>` in `item.ejs` also needs updating to send a POST request - add `method="post"` and (optionally) `action="/stuff/item/<%= id%>`:

```html
    <form method="post" action="/stuff/item/<%= id%>"> <!-- default action is the page's URL --> 
```

Again, the `action` attribute is optional here - the default action is the URL of the page, which matches the update to be done.

Once again, note the attribute `name` of the various input elements: "name", "quantity", and "description".

### (5.5.3) Handling an Update item POST request:

Now, let's write a route to receive the form's POST request and run the UPDATE query. 

```js
// define a route for item UPDATE
const update_item_sql = `
    UPDATE
        stuff
    SET
        item = ?,
        quantity = ?,
        description = ?
    WHERE
        id = ?
`
app.post("/stuff/item/:id", ( req, res ) => {
    db.execute(update_item_sql, [req.body.name, req.body.quantity, req.body.description, req.params.id], (error, results) => {
        if (error)
            res.status(500).send(error); //Internal Server Error
        else {
            res.redirect(`/stuff/item/${req.params.id}`);
        }
    });
})
```

Once again, let's break the code down and see how the Update operation flow works:

>```
> Browser --- request: POST URL -----> App Server
>                                      App Server --- UPDATE query ---> Database
>                                      App Server <-- confirmation ---- Database
> Browser <- response: REDIRECT------> App Server
>```

1. **The web server receives a POST HTTP request from a browser** - the user submits an HTML form. This POST handler receives the form data, which includes the values of the "name" and "quantity" and "description" inputs. These are accesible via `req.body.name`, `req.body.quantity` and `req.body.description`.
2. **The web server makes an UPDATE query to the database**. Those input values, along with `id` in the URL (`req.params.id`), are used to prepare and excute an `UPDATE` SQL statement, which changes the values in the row with matching `id` to the new the form values. 
3. **The web server waits for results of the query**; upon successful execution of the SQL,  `results` are returned from the database. However, there isn't much information that we're interested in beyond whether it was successful or if there was an error.
4. **The web server uses the query results to form and send the HTTP response back to the browser**. Once again, rather than generate a new page the server sends a "redirect" to the item detail page of the updated item.  In this case, `req.params.id` is used to construct the matching URL.
    > Again, the server could respond by rendering a special page with a message confirming the success of the update operation, but it makes sense to simply show the user the page that the operation has changed.


#### (5.4.3.1) Testing the Update operation:

To test, restart your server, navigate to `/stuff/:id` for some existing item, fill out the form inputs, and click Submit. You should be redirected to the same item detail page (a "refresh"), now updated with the data you entered. Return to the `/stuff` page and you should see the updated item there as well.

> You should notice in the server logs a `POST /stuff/:id` request, with a status code of `302` for redirect, followed by a `GET /stuff/:id` request.


> You can also see the POST request/response from your browser's Developer Tools; in the Network tab, watch for a request to `stuff` as you submit the Form. Click on it to view the details.
>
>The "Headers" sub-tab should show:
>>Request URL: http://localhost:8080/stuff/:id
>>Request Method: POST
>>Status Code: 302 Found
>
>And the "Payload" sub-tab should show:
>>name: ____
>>quantity: ____
>>description: ____
>
>displaying the values entered into the form.



### (5.5.4) Pre-populating the "Edit" form 

Lastly, it would be nice if the Edit form in `item.ejs` was already prepopulated with the current values of the data. This can be easily updated by adding `value` attributes to the input elements and using EJS to set them, much the way the rest of the page is rendered with data.

```html
<input type="text" name="name" id="nameInput" class="validate" data-length="32" value="<%= item%>" required>
...
<input type="number" name="quantity" id="quantityInput" value=<%= quantity%> required>
...
<input type="text" name="description" id="descriptionInput" data-length="100" value="<%= description%>">
```

Restart the server, open an item detail page, and confirm that the inputs in the form are now prepopulated.

## (5.6) What's next?

With all the basic CRUD routes implemented, we have a usable web-app! It is - *almost* - time to deploy a first version of our application to a cloud-hosting service. 

However, there are a smattering of improvements we should and can make before we put it online. In the next tutorial, we will implement and discuss these improvements.