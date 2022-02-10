---
layout: post
title: Simplifying CRUD operations
---

# Simplifying CRUD operations

The four basic database operations are Create, Read, Update and Delete (CRUD). 

Typically, applications using HTTP use the request method (GET, POST, PUT and DELETE are the most common) to refer to a specific CRUD operation:
-  GET method = Read
-  POST method = Create
-  PUT method = Update
-  DELETE method = Delete

We can reduce this set of operations to two: READ and WRITE. This would make writing applications easier and allow more flexibility with the communication protocol used. This goes well with PUB/SUB (Publish / Subscribe) communication protocols (like MQTT). In this case, publishing would be equivalent to WRITE and subscribing to READ.

The READ operation would be the same as the Read operation in CRUD, whereas the WRITE operation would replace Create, Update and Delete.

Some database engines already provide an "_upsert_" method (Update + Insert). The user provides a record with an _id_ and the database either updates the record with the same _id_ or inserts a new record if the _id_ is not found. This is a way of combining the Update and Create operations.

For example (this is pseudocode, where we call some READ and WRITE functions that represent the corresponding operations and we represent the records as json):
```
WRITE({id: 1, value: 11})
READ() -> [{id: 1, value: 11}]

WRITE({id: 2, value: 22})
READ() -> [{id: 1, value: 11}, {id: 2, value: 22}]

WRITE({id: 1, value: 0})
READ() -> [{id: 1, value: 0}, {id: 2, value: 22}]
```
In the above example, we inserted two records with _ids_ "1" and "2" and then updated the record with _id_ "1" using only the WRITE function.

Finally, if we want to delete a record, we can set a field called, for example, "_deleted_", that marks if the record has been deleted or not. This is a common practice when we want to keep all records even if the user "deletes" them from the front-end application. For instance, if an admin decides to delete a user, the back-end system will probably not remove the record for that user, but rather set a flag as "deleted". This would allow for later recovery, among other benefits.

This means that the Delete operation becomes an Update operation, where we update the _deleted_ field. We can do this again with our WRITE operation.

Continuing the above example:
```
WRITE({id: 1, deleted: true})
READ() -> [{id: 2, value: 22}]
```

The READ function ignores all the records that have the field "deleted" set to true.

With this, we have reduced the CRUD operation set to READ and WRITE.




