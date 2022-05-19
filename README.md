# Part 05: Implementing CRUD operations: Using Forms and POST requests

This tutorial follows after:
[Part 04: Connecting the Layers: Rendering templates with data](https://github.com/atcs-wang/inventory-webapp-04-connecting-layers-templates)

In this tutorial, we will continue to work across all 3 layers (Browser, Web Server, Database) to implement standard "CRUD" data operations for the inventory items.

Our app currently allows users to see (Read) data from the database, but not make any changes to the database; we will add functionality for the other three CRUD operations that cause database changes: Create, Update, and Delete.

## (4.1) The big picture for CRUD operations

CRUD is an acronym for the 4 basic data storage operations: Create, Read, Update, and Delete. Data-based applications typically provide one or more interactive ways for users to do each of these operations.

> Of course, in many cases, not *every* user should be allowed to do *every* kind of CRUD operation on *every* instance of data storage. We will explore ways to differentiate between different user *roles* and how to authorize specific operations later.

Recall from last tutorial that our web-app's "flow" generally follows this pattern: 

>```
> Browser --- request ---> App Server
>                          App Server --- query --> Database
>                          App Server <-- results ---- Database
> Browser <-- response --- App Server
>```

The exact details of how each of the 4 steps works can vary, but every operation we implement in this tutorial follows this pattern.


### Review: The flow of the Read operations

In the last tutorial, we already implemented 2 read operations - one for a specific item, one for the entire inventory - but its worth reviewing how it fits the 4-step pattern described above:

>```
> Browser --- request: GET URL ------> App Server
>                                      App Server --- SELECT query--> Database
>                                      App Server <-- results ARRAY ---- Database
> Browser <- response: RENDERED PAGE-> App Server
>```

1. **The web server receives a GET HTTP request from a browser** - either by entering a URL into their browser, or clicking a hyperlink
2. **The web server makes a SELECT query to the database**.
3. **The web server waits for results of the query**; if successful, the results are an *array* of objects representing a set of row entries.
4. **The web server uses the query results to form and send the HTTP response back to the browser**; a page is *rendered* from an EJS template using the query results.

Now, we can start to implement the other CRUD operations, and see how they are similar and different from the Read flow. 

Let's begin by writing SQL queries that will be used for the other CRUD operations. We'll put them in the subdirectory `db/queries/crud`. 

### (4.2) Writing SQL queries for CRUD operations

Each CRUD operation has a corresponding SQL command that is typically used in the "query" step of the flow:

| Operation | SQL Command | Description                                      |
|-----------|-------------|--------------------------------------------------|
| Create    | INSERT      | Add new entry/entries                            |
| Read      | SELECT      | Retrieve, search, or view existing entry/entries |
| Update    | UPDATE      | Edit existing entry/entries                      |
| Delete    | DELETE      | Remove existing entry/entries                    |


#### (4.2.1)  INSERT query (Create):

We actually wrote some INSERT queries in part 3 of the tutorials; 

For the **Create** operation, `insert_stuff.sql` adds a new entry to the `stuff` table, given initial `item` and `quantity` values. 
No `id` is specified; the database will automatically determine a unique `id` for the new element.

```sql
INSERT INTO stuff
    (item, quantity)
VALUES
    (?, ?)
```

If successful, this operation returns the `id` (primary key) of the newly inserted entry.

#### (4.2.2)  UPDATE query:
For the **Update** operation, `update_stuff.sql` updates the `item`, `quantity`, and `description` values for an entry from the `stuff` table, matching a given `id`:

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

This operation doesn't really return data, although it does provide confirmation of how many rows get updated. (Since the `id` is a primary key, it should be just 1 or 0)


#### (4.2.3)  DELETE query:

For the **Delete** operation, `delete_stuff.sql` deletes an entry from the `stuff` table, matching a given `id`:

```sql
DELETE 
FROM
    stuff
WHERE
    id = ?
```

This operation doesn't really return data, although it does provide confirmation of how many rows get deleted. (Since the `id` is a primary key, it should be just 1 or 0)



### (4.3) Implementing the Delete Operation

Although it is listed last among the CRUD operations, let's begin by implementing the **Delete** functionality, because it is most similar to the flow of the Read operation. 

Similar to how our app interprets to GET requests to the URL `/stuff/item/:id` as a Read operation for the entry with given `id`, our app will interpret GET requests to the URL `stuff/item/:id/delete` as a Delete operation for the entry with given `id`.

### (4.3.1) Handling a Delete GET request:

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
>                                      App Server --- SELECT query--> Database
>                                      App Server <-- results ARRAY ---- Database
> Browser <- response: RENDERED PAGE-> App Server
>```

1. **The web server receives a GET HTTP request from a browser** - either by entering a URL into their browser, or clicking a hyperlink.
2. **The web server makes a DELETE query to the database**, using the `id` in the URL to complete the query.
3. **The web server waits for results of the query**; the contents of the results returned are not particularly important, but we want to know when the request is complete and that there was no error.
4. **The web server uses the query results to form and send the HTTP response back to the browser**; Rather than render a new page, the server sends a "redirect" that navigates the user back to the homepage to see. 

Now test out the handler by entering URLs into your browser of the format `/stuff/item/:id/delete`, replacing `:id` with the `id` values of entries in the database's `stuff` table (e.g. 1, 2, etc.).

> If we wanted, we could instead render a special response/page that gives the user confirmation of their successful deletion. However, being redirected immediately to the inventory page where the user can visually confirm the removal of the item also communicates the success of the operation.

Re-run your `db_init.js` script to re-set your database afterwards.

```
> node db/db_init.js
```

> Using a hyperlink to produce a GET request is a functional way to implement deletion, but is a little strange conceptually. 
> GET requests are generally supposed to be for *accessing* data / web pages, 
> not requesting changes to data or the pages. We think of hyperlink URLs as representing a resource to be seen, not an action to perform.
> 
> More recently, with the advent of client-side rendered (CSR) applications, there are alternative ways to doing this. We'll explore these much later.


### (4.3.2) Making the "Delete" button send a GET request

We don't really intend for users to type raw URLs into their browser to trigger a deletion request; rather, we can create buttons that are actually hyperlinks which link to that.

In both the inventory page (template `stuff.ejs`) and the item detail page (template `item.ejs`), there are nonfunctional "Delete" buttons. 

Recall that, in the previous tutorial, we gave the `href` attribute of each of the "Edit/Info" buttons on the inventory page a unique URL, composed from the `id` of the corresponding item/row (`inventory[i]`), which goes to specific item detail page for that item: 

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

As a result, each "Edit/Info" button on each table row was associated with a unique URL. Clicking the button to "follow the link" sends a GET request to the app server.

In a very similar manner, we can also give the adjacent "Delete" buttons in `stuff.ejs` unique URLs based on the same `id`: 

```ejs
    <a class="btn-small waves-effect waves-light red" href="/stuff/item/<%= inventory[i].id %>/delete">
        <i class="material-icons right">delete</i>
        Delete
    </a>
```

The Delete button on the item detail page (template `item.ejs`) needs to similarly link to a URL which matches the `id` of the data being rendered on that page: 

```ejs
    <a class="btn-large waves-effect waves-light red right"  href="/stuff/item/<%= id %>/delete">
        <i class="material-icons right">delete</i>Delete
    </a>
```

Confirm that clicking these updated Delete buttons on both pages triggers GET requests for URLs of the pattern `/stuff/item/:id/delete`, where `:id` matches the `id` of the data, and that you are redirected back to the `/stuff` page.

>The server logs should report a status code of 302 (REDIRECT) for `GET /stuff/item/:id/delete`, followed by a `GET /stuff`.


#### (4.4) Overview: Using Forms and POST requests

So far, all of the interaction between the user and the web server have been through GET requests sent by the browser. These are typically triggered by

1. A URL being entered into the browser's URL bar
2. A hyperlink being clicked
3. A link to a resource on an HTML page (e.g. CSS, images, etc.) 
4. Being redirected to a URL by the server

In addition to GET requests, which are typically associated with *downloading* data, browsers can also send POST requests, which are typically are used for *uploading* data. 

Most commonly, POST requests are triggered via **HTML forms**, where users enter values into various inputs, then "Submit" the contents of the form. 

To make an HTML `<form>` element that sends a POST request upon Submit, its HTML attribute `method` should be set to `"post"`. 

Here is a simple example of a form designed to send a POST request:

```html
<form method="post" action="/sample/url">
    <input type="text" name="name">
    <input type="number" name="age">
    <button type="submit">
</form>
```

A few more details about standard form usage:

- Optionally, the attribute `action` can be set to the URL to which the post requests should be sent.  If `action` is not defined, the form POSTs to the same URL of the current page.

- It is important that each `<input>` element in the form includes a `name` attribute, which describes the meaning of the input.  

- A `<button>` element with attribute `type="submit"` is usually included in a form too; clicking this button triggers the POST request.

Fortunately, our prototypes already had two forms set up - the only thing that needs to be changed is their `method` must be set to "post". Then, we just need to set up POST routes (which work similarly to GET routes) on the server to handle the requests. See the sections below for step by step instructions.

>NOTE: If the `method` is left undefined (like our prototype's forms were) or explictly set to `"get"`, HTML forms can also send GET requests - the form will construct a special URL from the input values, acting like a customizable hyperlink. 
> If you click submit on our app's forms now, they appear to refresh the page. But if you look at the URL bar, the new URL includes a `?` symbol, followed by the form's inputs. Express typically ignores that part of the URL when resolving the matching GET route path, but they can be treated as **parameters**. **Search bars** are a common application of this kind of form and parameterized GET request; we will add one later!

### (4.4.1) Configuring Express to parse POST request bodies

// TODO: Explain URL-embedded vs multipart

In order for Express to handle POST requests more easily, we need to add some middleware. 

Add this line to the middleware section (around other `app.use` lines) in `app.js`
```js
// Configure Express to parse URL-encoded POST request bodies (traditional forms)
app.use( express.urlencoded({ extended: false }) );
```

To handle a post request to the URL `/stuff`, we can use a route definition like this:
```js
app.post("/stuff", ( req, res ) => {
    //access to form values via req.body's properties:
    // For example:
    // req.body.name
    // req.body.quantity
    ...
}
```

### (4.4.2) Implementing the Create operation 

#### (4.4.2.1) Sending an "Add" POST request: - the "Add" form

Let's make the Add Stuff form on the `/stuff' page work!

Update the form in the `stuff.ejs` template:

```html
    <form method="post" action="/stuff"> 
```
Again, the `action` is actually not necessary to be defined explicitly; by default, the action is the URL of the current page.

Note the attribute `name` of the various input elements:
```html
    <input type="text" name="name" id="nameInput" class="validate" data-length="32" required>
    ...
    <input type="number" name="quantity" id="quantityInput" value=1 required>
```

Note that the form inputs are named "name" and "quantity".

#### Handling an "Add" POST request:

Add this code to your `app.js`, below the other routes.
```js
// define a route for item CREATE
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

1. This POST handler receives the form data, which includes the values of the "name" and "quantity" inputs. These are accesible via `req.body.name` and `req.body.quantity`.

2. Those input values are used to prepare and excute an `INSERT` SQL statement, which adds a new row to the `stuff` table with the form values. 

3. Upon successful execution of the SQL, the `results` returned from the database includes `insertId`, the value of the primary key for the newly inserted row.  

4. The handler then (finally) responds to the HTTP request a "redirect". (HTTP status code 302) - essentially, an instruction to navigate to (that is, send *another* GET request) a different URL. In this case, `results.insertId` is used to redirect to the item detail page of the newly added row. 

### (4.5) Implementing the Update operation - 

#### Sending an Update POST request: - the "Edit" form

The "Edit" form in `item.ejs` needs updating:

```html
    <form method="post" action="/stuff/item/<%= id%>"> <!-- default action is the page's URL --> 
```

Again, the `action` attribute is optional here - the default action is the URL of the page, which matches the update to be done.

Once again, note the attribute `name` of the various input elements: "name", "quantity", and "description".


#### Handling an "Update" POST request:

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


1. This POST handler receives the form data, which includes the values of the "name" and "quantity" and "description" inputs. These are accesible via `req.body.name`, `req.body.quantity` and `req.body.description`.

2. Those input values, along with `id` in the URL (`req.params.id`), are used to prepare and excute an `UPDATE` SQL statement, which changes the values in the row with matching `id` to the new the form values. 
 
3. and 4. Upon successful execution of the SQL, the handler then (finally) responds to the HTTP request a "redirect" (HTTP status code 302) - to  a different URL. In this case, `req.params.id` is used to redirect to the item detail page of the changed row.  

#### (4.5.1) Pre-populating the "Edit" form 

It would be nice if the Edit form already contained the current values of the data. This can be easily updated by adding `value` attributes and using EJS to set them with the data context, much the way the rest of the page includes the data.

```html
<input type="text" name="name" id="nameInput" class="validate" data-length="32" value="<%= item%>" required>
...
<input type="number" name="quantity" id="quantityInput" value=<%= quantity%> required>
...
<input type="text" name="description" id="descriptionInput" data-length="100" value="<%= description%>">
```


## Summary of CRUD operations and routes


// TODO