# API calls and responses the front-end expects to be able to make and receive

## List of APIs the current version of the application expects

1. [/user/:id/history](#/user/:id/history)
2. [/config](#/config)
3. [/sets](#/sets)
  a. [/sets/groupby](#/sets/groupby)
4. [/entities](#/entities)
  b. [/entities/:id/flag](#/entities/:id/flag)
  c. [/entities/:id/ban](#/entities/:id/ban)
  d. [/entities/:id/notes](#/entities/:id/notes)
  e. [/entities/:id/notes/:id](#/entities/:id/notes/:id)
5. [/documents](#/documents)
  b. [/documents/:id/flag](#/documents/:id/flag)
  c. [/documents/:id/ban](#/documents/:id/ban)
  d. [/documents/:id/notes](#/documents/:id/notes)
  e. [/documents/:id/notes/:id](#/documents/:id/notes/:id)
6. [Other set types](#other-set-types)

## API Details

### /user/:id/history
#### Summary
Activity history is currently the only piece of user data the application needs to function. It has
no other concept of a user for any other data. 

#### GET
On load the App component makes a request to get the users history. Currently the :id is ignored due
to the server just pulling data from a local JSON file. In response the application expects to get
an array of history objects.

##### Expected Query String Supported
None

##### Expected Response 
```json
  [
    {
      "action": "an action taken by the user",
      "actor": "The username associated with the action",
      "date": "a Date string in the format: June 17, 2018 21:12:32"
    }
  ]
```

#### PUT
When a recordable action is performed by the user, the application saves that action to state and 
then attempts to update the user's history on the server by sending the whole history from the local
state. The body of the PUT will be an array of history objects (as above). The client expects no
response body currently, but a success/fail with error text would allow for an error message to be 
displayed.

##### Prefered Response
Success
```json
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```
### /config
#### Summary
The configuration for each module of the application. This file dictates which data is displayed in
each SetViewer mode as well as which values are available as fitler factors and entity types while
creating sets. Each SetViewer mode section is broken into three arrays: summary, details, and 
notShown. The summary section controls the data shown in the summary cards listed in the left column,
the details section controls the data shown in the larger details viewer on the right side of the 
component, and the notShown list is currently ignored -- it exists as a helper to humans editing the
JSON by hand to know which values are available to them. 

#### GET
On load the App component makes a request to get the configuration file that it uses in loading all 
the other components that make up the application. 

##### Expected Query String Supported
None

##### Expected Response
```javascript
  {
    "dashboard": {
      "setTypes": [
        "an array of strings representing set types available"
      ],
      "filterValues": [
        "an array of strings representing available fields to filter on in the new set builder"
      ]
    },
    "entitiesDataView": {
      "fetchLimit": 30, // an integer value passed to /entities GET requests as the limit query param
      "readInterval": 2000, // an interval in ms to control mark as read
      "headlineElement": "a string representing the data field to use as a headline element",
      "summary": [
        "an array of strings representing the data fields to show in the summary card"
      ],
      "details": [
        "an array of strings represnting the data fields to show in the details view"
      ],
      "notShown": [
        "an array of strings representing the data fields not in use. This is not currently used by the system, it is simply included as a helper to humans editing the file manually"
      ]
    },
    "documentsDataView": {
      "fetchLimit": 30, // an integer value passed to /documents GET requests as the limit query param
      "readInterval": 2000, // an interval in ms to control mark as read
      "headlineElement": "a string representing the data field to use as a headline element",
      "summary": [
        "an array of strings representing the data fields to show in the summary card"
      ],
      "details": [
        "an array of strings represnting the data fields to show in the details view"
      ],
      "notShown": [
        "an array of strings representing the data fields not in use. This is not currently used by the system, it is simply included as a helper to humans editing the file manually"
      ]
    }
  }
```

### /sets
#### Summary
The set objects act as keys for all of the other components to make requests for the data contained 
in them in addition to being displayed by the DashboardView component. Sets do not contain any 
actual set data, only metadata about the set used to to help a user select a set or create new sets.

#### GET
On load the App component makes a requests to get the list of sets available to the user.

##### Expected Query String Supported
none

##### Expected Response
```javascript
  {
    "the set's id": {
      "setName": "a string representing the user generated name of the set",
      "id": 123456,
      "setBelongsTo": "either 'user' or 'team' to identify to the client which group the set is in",
      "type": "a string value that confoms to one of the setTypes passed in /config",
      "filter": "a filter string used by the FilterBuilder representing the last filter applied by this user",
      "sort": [
        "a sort array used by the SortControler representing the last sort applied by this user"
      ],
      "groupByFields" : [ "An array of strings representing the Enum values"],
      "inGroup": "string representing the group this set belongs to or null if it's not in a group",
      "history": [
        {
          "action": "an action taken by the user",
          "actor": "The username associated with the action",
          "date": "a Date string in the format: June 17, 2018 21:12:32"
        }
      ],
      "stats": {
        "count": 12345 // the number of items in the set,
        "createdOn": "a Date string in the format: December 24, 2017 00:24:00",
        "createdBy": "A user name",
        "lastViewed": "a Date string in the format: June 17, 2018 21:12:32",
        "lastViewedBy": "A user name"
      }
    }
  }
```

#### POST
On create by either an operation on existing set(s) or by filtering the whole corpus of data, the 
SetCreator component makes a request to get a new set from the server. The SetCreator component waits
until the server responds before it updates the local state with the response body.

##### Request body will be as follows
```javascript
  // NOTE: this is probably not the most restful way of doing this, but it is less complex than 
  //       having the client differentiate between operations it's not actually performing.
  //       Logic will need to be added to the client if it is decided to break this into muktiple
  //       endpoints. 
  {
    "operation": "A string representing the operation to construct the set. Will always be one of: 'join', 'intersect','difference', or 'filter'",
    "setName": "a string representing the user generated name for the set",
    // sets are the sets to be used in the generation of a new set. 
    // If the operation is 'filter' the sets array will always be null or empty
    // If the operation is a unary operation (negate or random) sets will have a length of 1
    // For all other operations the sets array will have a length of 2
    "sets": [ 
      {
        "id": 12345, // an id for the set to use in the above selected operation
        "negate": true, // a flag representing if the negate operation should be performed on this set
        "random": 1345 // an integer representing how many random elements of this set should be used
        // if random: 0 should be interpreted as 'false' and the entire set should be used
      }
    ],
    //If the operation is not 'filter', this field should be null or undefined
    //If 'operation' is filter, this is a string representing a filter query constructed as follows: 
    // a data field + an operation + some user entered value + AND/OR/NOT(iff an aditional filter criterion exists)
    // the string includes a single space inbetween each element in the above formula.
    "filter":'data_prop > user_value AND data_prop = user_value',
    // This should be either 'user' or 'team'. If 'user', then we assume set belongs to user with specified 'userName'
    "setBelongsTo": 'user',
    //user name, mandatory
    "userName": 'john',
    "type":"a string value that confoms to one of the setTypes passed in /config",
    "groupByFields":["array","of","fields","to","group","by"]
  }
```

##### Expected Response
Success 
```javascript
  {
    "success": true,
    "body": {
      "setName": "a string representing the user generated name of the set",
      "id": 123456,
      "setBelongsTo": "either 'user' or 'team' to identify to the client which group the set is in",
      "type": "a string value that confoms to one of the setTypes passed in /config",
      "filter": "a filter string used by the FilterBuilder representing the last filter applied by this user",
      "sort": [
        "a sort array used by the SortControler representing the last sort applied by this user"
      ],
      "groupByFields" : [ "An array of strings representing the Enum values"],
      "inGroup": "string representing the group this set belongs to or null if it's not in a group",
      "history": [
        {
          "action": "an action taken by the user",
          "actor": "The username associated with the action",
          "date": "a Date string in the format: June 17, 2018 21:12:32"
        }
      ],
      "stats": {
        "count": 12345 // the number of items in the set,
        "createdOn": "a Date string in the format: December 24, 2017 00:24:00",
        "createdBy": "A user name",
        "lastViewed": "a Date string in the format: June 17, 2018 21:12:32",
        "lastViewedBy": "A user name"
      }
    }
  }  
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```
#### PUT
When the set name field in the DashboardView component loses focus the DashboardView makes a request
to update the set by sending the updated information. The client expects the endpoint to be more 
open-ended than that though, sending an object with the items to be  updated that could be any of 
the items in the set object. On success the client shallow merges these updates into it's version of
the set in state.


##### Request body will be as follows
NOTE: this object could contain any number of elements from the set obect. Currently the application
      is only making one request of this endpoint to update a set's name.
```json
  {
    "setName": "some string representing a new name"
  }
```

##### Expected Response
Success
```json
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

#### DELETE
The DashboardView component makes a request against this endpoint whenever a user clicks the delete
set button. The client only allows sets in the user group to be deleted, so verifications of that 
permission are not required. The client waits for the response before updating it's own state. 

##### Request body will be as follows
```json
  {
    "setId": 12345 
  }
```

##### Expected Response
Success
```json
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /sets/groupby
#### Summary
This endpoint is distinct from the /sets endpoint because it should return an array of sets instead
of just a single set.

#### POST
The DashboardView component makes a request against this endpoint whenever the user confirms a 
groupBy operation on the selected set. 

##### Request body will be as follows
```javascript
  {
    "id": 123456 // a set id for the set to perform the groupby operation on
    "groupBy": "a string representing the ENUM in the set to groupby", 
    "groupName": "a string repreenting the name the user has chosen for the group to be created"
  }
```

##### Expected Response
Success
```javascript
  {
    "success": true,
    "body": [
      {
        "setName": "a string representing the user generated name of the set",
        "id": 123456,
        "setBelongsTo": "either 'user' or 'team' to identify to the client which group the set is in",
        "type": "a string value that confoms to one of the setTypes passed in /config",
        "filter": "a filter string used by the FilterBuilder representing the last filter applied by this user",
        "sort": [
          "a sort array used by the SortControler representing the last sort applied by this user"
        ],
        "groupByFields" : [ "An array of strings representing the Enum values"],
        "inGroup": "string representing the group this set belongs to or null if it's not in a group",
        "history": [
          {
            "action": "an action taken by the user",
            "actor": "The username associated with the action",
            "date": "a Date string in the format: June 17, 2018 21:12:32"
          }
        ],
        "stats": {
          "count": 12345 // the number of items in the set,
          "createdOn": "a Date string in the format: December 24, 2017 00:24:00",
          "createdBy": "A user name",
          "lastViewed": "a Date string in the format: June 17, 2018 21:12:32",
          "lastViewedBy": "A user name"
        }
      }
    ]
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /entities
#### Summary
This endpoint is the only way the application gets the actual data from a set of entities. T

#### GET
The SetViewer component makes calls against this endpoint with various query params when the set it's
passed has a type of "entities"

##### Expected Query String Supported

- limit=1234 : 
  an integer configured by the user to limit the number of set items returned
- offset=1234 : 
  an integer provided to make subsequent calls for the next "page" of data
- sort[]='data_prop.asc'&sort[1]='data_prop.desc' : 
  an array of data fields to sort by and the direction of the sort. The array is in order of the 
  priority of the sort (e.g. sort by factor 0 asc THEN by factor 1 desc)
- fields='field1;field2;...' :
  fields to return, separated by semi-colon. For instance, specifying 'notes;emails;toCount'
  would exclude all fields other than these three.
  If not provided, assume that ALL fields are returned. 
- filter='data_prop > user_value AND data_prop = user_value" :
  a string representing a sort constructed as follows: 
    a data field + an operation + some user entered value + AND/OR/NOT(iff an aditional filter 
    criterion exists)
  the string includes a single space inbetween each element in the above formula.

##### Expected Response
```javascript
  [
    {
      //EmbryoEntity documents that are linked to this Entity. 
      //Given Entity e, each key/value pair in this object corresponds to one EmbryoEntity from 
      // db.EmbryoEntity.find({_id:{$in:e.embryoEntitiesIds}})
      "embryos": {
        "some embryo id":{
          // this follos the EmbryoEntity schema. The client application has a helper method to render 
          // the current list of elements in this object that are listed in the /config response
          // but will render any values that are in this object except for another object
        },
        
        //Given Entity e, this is derived by taking emailAddress.emailAddress for 
        //  each emailAddress in e.emailAddresses. 
        "emails":[
          "an array of strings representing all the email addresses associated with the entity"
        ],
        //EntityNote documents that are linked to this Entity.
        //Given Entity e, each key/value pair in this object corresponds to one EntityNote from
        // db.EntityNote.find({entityId:e._id})
        "notes":{
          "some note id":{
            "entityId":"an entity id for the entity the note belongs to",
            "note":" a note body",
            "createdAt":"2018-06-11T21:04:42.759Z",
            "updatedAt":"2018-06-11T21:04:42.759Z"
          }
        },
        //The following three fields come from collection EntityFlag. Given Entity e,
        // and given en = db.EntityFlag.findOne({_id:e._id}),
        // these come from en.flagged, en.banned, and en.read 
        "flagged":false, // a Boolean represneting if the entity is flagged
        "banned":false, // 
        "read": false // a Boolean representing if the user has viewed this item
        
        //Ignore this, for now...
        "documents":{
          "some document id":{
            "embryo_id":"an embryo id for the embryo the document belongs to",
            "relationship":"FROM" // an Enum conforming to the documents schema 
          }
        },
        
        //If e.entityRole is not null, take from e.entityRole.roleConfidence. Null otherwise.
        "roleConfidence":null, 
        //If e.entityRole is not null, take from e.entityRole.source. Null otherwise. 
        "roleSource":null, 
        // If e.entityRole is not null, take from e.entityRole.status. Null otherwise. 
        "roleStatus":null, 
        //If e.entityRole is not null, take from e.entityRole.entityRoleType. Null otherwise. 
        "roleType":null, 
        //If e.primaryName is not null, take from e.primaryName.name. Null otherwise. 
        "primaryName":"a string representing the most likely name for the entity",
        //Take from e._id
        "id":"an id for the entity itself",
        //e.numEmailsSent
        "numEmailsSent":0, 
        //e.numEmailsFrom
        "fromCount":5, 
        //e.nameVariants
        "nameVariants":[
          "an array of strings representing all the possible other names the entity could have"
        ],
    }
  ]
```

### /entities/:id/flag
#### Summary
This endpoint is called by the SetViewer to set or remove the flagged state of a specific entity
#### POST
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

#### DELETE
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /entities/:id/ban
#### Summary
This endpoint is called by the SetViewer to set or remove the banned state of a specific entity
#### POST
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

#### DELETE
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /entities/:id/notes
#### Summary
This endpoint is used solely for the creation of a new note attatched to an entity itself, not one
of it's embryos
#### POST
The SetViewer component makes a call to this endpoint when the user presses the save button on the 
notes editing UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
a UTF-8 string that contains the body of the note

##### Expected Response

Success
```JSON
  {
    "success": true,
    "body": {
      "entityId":"an entity id for the entity the note belongs to",
      "note":" a note body",
      "createdAt":"2018-06-11T21:04:42.759Z",
      "updatedAt":"2018-06-11T21:04:42.759Z"
    }
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /entities/:id/notes/:id
#### Summary
This endpoint is used to update and remove notes attached to an entity itself, not one of it's embryos
#### PUT
The SetViewer component makes a call to this endpoint when the user presses the save button on the 
notes editing UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
a UTF-8 string that contains the body of the note

##### Expected Response

Success
```JSON
  {
    "success": true,
    "body": {
      "entityId":"an entity id for the entity the note belongs to",
      "note":" a note body",
      "createdAt":"2018-06-11T21:04:42.759Z",
      "updatedAt":"2018-06-11T21:04:42.759Z"
    }
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```
#### DELETE
The SetViewer component makes a call to this endpoint when the user presses the delete button on the
notes viewer UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /documents
#### Summary
This endpoint is the only way the application gets the actual data from a set of documents.

#### GET
The SetViewer component makes calls against this endpoint with various query params when the set it's
passed has a type of "documents"

##### Expected Query String Supported

- limit=1234 : 
  an integer configured by the user to limit the number of set items returned
- offset=1234 : 
  an integer provided to make subsequent calls for the next "page" of data
- sort[]='data_prop.asc'&sort[1]='data_prop.desc' : 
  an array of data fields to sort by and the direction of the sort. The array is in order of the 
  priority of the sort (e.g. sort by factor 0 asc THEN by factor 1 desc)

- filter='data_prop > user_value AND data_prop = user_value" :
  a string representing a sort constructed as follows: 
    a data field + an operation + some user entered value + AND/OR/NOT(iff an aditional filter 
    criterion exists)
  the string includes a single space inbetween each element in the above formula.

##### Expected Response
```javascript
  [
    {
      "id": "this document's id",
      "docId": "this document's doc_id",
      "flagged":false, // a Boolean represneting if the entity is flagged
      "banned":false, // a Boolean represneting if the entity is banned
      "read": false, // a Boolean representing if the user has viewed this item
      "emails": [
        {
          "content": "a string representing the body of the document",
          "to": [
            {
              "primaryName" : "the name of the entity",
              "id" : "an entity id for this entity",
              "email" : "the email address used by this entity for this document"
            }
          ],
          "from": [
            {
              "primaryName" : "the name of the entity",
              "id" : "an entity id for this entity",
              "email" : "the email address used by this entity for this document"
            }
          ],
          "cc": [
            {
              "primaryName" : "the name of the entity",
              "id" : "an entity id for this entity",
              "email" : "the email address used by this entity for this document"
            }
          ],
          "bcc": [
            {
              "primaryName" : "the name of the entity",
              "id" : "an entity id for this entity",
              "email" : "the email address used by this entity for this document"
            }
          ],
          "subject": "the subject value for this message",
          "spans": [
            {
              "start": 2049, // the character this span begins at
              "end": 2349, // the character this span ends at
              // the type of span this is. The client accepts "DISCLAIMER", "ENTITY", and "SCORE"
              "type": "DISCLAIMER", 
              // NOTE: there is currently no rendering logic in the client to handle anything other
              //       than a string in spanInfo as no examples of other values were provided 
              "spanInfo": null // an object representing some span info or null
            },
          ],
          "attachments": [ 
            {
              "docId" : "the doc_id for the attached document",
              "id" :  "the id of the attached document"
            }
          ]
        }
      ]
    }
  ]
```

### /documents/:id/flag
#### Summary
This endpoint is called by the SetViewer to set or remove the flagged state of a specific document
#### POST
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

#### DELETE
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /documents/:id/ban
#### Summary
This endpoint is called by the SetViewer to set or remove the banned state of a specific entity
#### POST
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

#### DELETE
The client waits on the response from the server to update it's local state
##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /documents/:id/notes
#### Summary
This endpoint is used solely for the creation of a new note attatched to a document
#### POST
The SetViewer component makes a call to this endpoint when the user presses the save button on the 
notes editing UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
a UTF-8 string that contains the body of the note

##### Expected Response

Success
```JSON
  {
    "success": true,
    "body": {
      "entityId":"an entity id for the entity the note belongs to",
      "note":" a note body",
      "createdAt":"2018-06-11T21:04:42.759Z",
      "updatedAt":"2018-06-11T21:04:42.759Z"
    }
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### /documents/:id/notes/:id
#### Summary
This endpoint is used to update and remove notes attached to adocument
#### PUT
The SetViewer component makes a call to this endpoint when the user presses the save button on the 
notes editing UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
a UTF-8 string that contains the body of the note

##### Expected Response

Success
```JSON
  {
    "success": true,
    "body": {
      "entityId":"an entity id for the entity the note belongs to",
      "note":" a note body",
      "createdAt":"2018-06-11T21:04:42.759Z",
      "updatedAt":"2018-06-11T21:04:42.759Z"
    }
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```
#### DELETE
The SetViewer component makes a call to this endpoint when the user presses the delete button on the
notes viewer UI. The component waits for the server response before saving it's local state.

##### Request body will be as follows
the body will be empty

##### Expected Response

Success
```JSON
  {
    "success": true
  }
```

Error
```json
  {
    "success": false,
    "error": "useful text explaining the error and how to resolve it if possible"
  }
```

### Other set types
The client does not make a distinction of types of sets beyond the shape of the objects that get
handed to it. Internally the client uses either 'entities' or 'documents' to decide which components
it should use to render the information it receives. The client can render any set of data as long
as it is shaped either like the entity or document objects. More generally:

#### Entity type objects
An entities type object can have any any number of properties at it's first level. Any property
that is an object will be iterated through and it's results displayed in the ComplexDetailList
component which expects the object it is passed to behave like a dictonary. Any property that is an
array will be itterated through and reduced to a string of comma seperated values.
Each key in the object will be used as the label for each value. 

```javascript
  {
    // this object will be rendered in the ComplexDetailsList
    "some dictionary like object": {
      "some nested array":[],   // this will be rendered as a comma seperrated list 
    },
    // these will all be rendered as strings
    "some array": [], //this will be rendered as a comma seperated list
    "some Number": 123435, 
    "some String": "some value"
    "some Boolean": true 
  }
```

#### Document type objects
A documentes type object can have any number of peroperties at it's first level. Any property that
is an array will be passed to the DocumentViewer component which expects it to have email type
objects inside it. A document like object should render any parts of the emails object that it has 
and igore the missing pieces. 

```javascript
  {
      // these will all be rendered as strings
    "some Number": 123435, 
    "some String": "some value"
    "some Boolean": true 
    // the below are all rendered by the Document viewer. The key of the array is used as the
    // label for the focument viewer
    "emails": [
      {
        "content": "a string representing the body of the document",
        "to": [
          {
            "primaryName" : "the name of the entity",
            "id" : "an entity id for this entity",
            "email" : "the email address used by this entity for this document"
          }
        ],
        "from": [
          {
            "primaryName" : "the name of the entity",
            "id" : "an entity id for this entity",
            "email" : "the email address used by this entity for this document"
          }
        ],
        "cc": [
          {
            "primaryName" : "the name of the entity",
            "id" : "an entity id for this entity",
            "email" : "the email address used by this entity for this document"
          }
        ],
        "bcc": [
          {
            "primaryName" : "the name of the entity",
            "id" : "an entity id for this entity",
            "email" : "the email address used by this entity for this document"
          }
        ],
        "subject": "the subject value for this message",
        "spans": [
          {
            "start": 2049, // the character this span begins at
            "end": 2349, // the character this span ends at
            // the type of span this is. The client accepts "DISCLAIMER", "ENTITY", and "SCORE"
            "type": "DISCLAIMER", 
            // NOTE: there is currently no rendering logic in the client to handle anything other
            //       than a string in spanInfo as no examples of other values were provided 
            "spanInfo": null // an object representing some span info or null
          },
        ],
        "attachments": [ 
          {
            "docId" : "the doc_id for the attached document",
            "id" :  "the id of the attached document"
          }
        ]
      }
    ]
  }
```
