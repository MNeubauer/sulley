# Sulley for MongoDB
A fuzzer compatible with the MongoDB wire protocol

## Overview
This tool is a fuzz tester for the MongoDB wire protocol. A user can create messages that will be iteratively modified, mutated, and subsequently sent to a mongod server. This iterative process is handled mainly by internals of the Sulley fuzzing framework. Communicating with the framework is programatic and relatively simple. To use this tool for actual MongoDB testing, see the [WireFuzz github repository](https://github.com/10gen-interns/fuzzing/tree/matt/wirefuzz) and ensure that the requirements.txt file in that repo contains the correct branch of this repo.


## Sulley
* This tool was built on the Sulley Fuzzer, for detailed information see the following pages
    - For the official sulley readme, see [SULLEY.md](./SULLEY.md)
    - For the tutorial/manual, see the [Sulley Manual](http://www.fuzzing.org/wp-content/SulleyManual.pdf)

* The Basics
    - **Primitives** are the lowest level item in Sulley. They are used to describe different types of simple data objects such as integers, strings, static data, and Sulley generated random data.
    - **Legos** are user defined primitives, more on this faculty later as it is heavily relied on for testing the MongoDB wire protocol.
    - **Blocks** are groups of primitives. Blocks are also primitives, and can therefore be nested within each other.
    - **Requests** are a sequence of blocks and primitives that represent one part of a conversation between Sulley and a server.
    - **Sessions** are a graph of requests, that constitute one or more full conversations between Sulley and a server.
    - Sulley also has some tools available for post mortem analysis and test case replay. For details on what is supported see the [Post Mortem Tools](https://github.com/10gen-interns/fuzzing/blob/matt/wirefuzz/README.md#post-mortem-tools) section of the README.md dedicated to testing the MongoDB wire protocol.


## Getting Started
### Dependencies
See [requirements.txt](./requirements.txt)  
To install dependencies run `pip install -r requirements.txt`  
 - pymongo
    - Testing currently depends on pymongo for its bson module

## Design
* The purpose of this project is to allow developers and testers a way to easily send (mostly) well formed MongoDB wire messages to a mongod server. This is accomplished by having an interface that allows users to specify the intuitive content of the message without being conerned with low level details such as bit ordering or Sulley internals.

* One of the main reasons Sulley was selected as the framework for fuzzing the MongoDB wire was that the user's code is written in a programming language (python) and can take advantage of its facilities.

* **Legos** take advantage of Python's facilities and their use encourage a programatic way of representing wire messages that encourages code reuse - especially via inheritance. Each MongoDB command can be represented as its own lego which can be found in [sulley/legos/mongo.py](./sulley/legos/mongo.py).
    - All legos extend Sulley's [block](./sulley/blocks.py) class. The [MongoMsg](./sulley/legos/MongoMsg.py) class extends the block class and is a base class for legos that represent MongoDB messages. MongoMsg has a few main purposes:
        - Create the MsgHeader for each message
        - Hide some repeated code making it easier to read the code in its subclasses
        - Wrap simple lines of code if they are planned to become more complex in the future
            - e.g. Making the BSON interpretation more complex

## Usage
### Requests
* The Sulley definition of a call for a lego is `def s_lego(lego_type, value=None, options={})`
* All `lego_type`s can be found in [sulley/legos/\__init\__.py](sulley/legos/__init__.py)
* Calls to s_lego for MongoDB messages use the options dict as a way of passing initial values for the message.
* See the [MongoDB wire protocol spec](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/) for details on what is expected for each message.
    - Notes on the **MsgHeader**:
        - The messageLength field is calculated by a sizer upon initialization of each lego and should not be specified
        - The opCode field is implied by the type of lego that is called and should also not be specified
        - It is suggested that the user not specify the responseTo field
            - SSL handshake is expected if this field is not 0 or -1
            - The fuzzer will test across both of these values if the field is not specified
    - The `fullCollectionName` field is not expected. Instead pass separate `db` and `collection` fields wherever the spec calls for `fullCollectionName`. This is so Sulley can fuzz the delimiter properly.
    - Legos currently do not expect options for fields that are reserved and filled with zeros.
* Multiple requests can be made per file in the [requests/](./requests) directory.
* Each request starts with `s_initialize("example request")`.
    - A session uses this request as a node with a call to `sess.connect("example request")`.

* An example request containing one insert message:
```python
s_initialize("insert")
s_lego("OP_INSERT", options=
    {
        # MsgHeader
        "requestID": 98134,
        # OP_INSERT
        "flags": 1,
        "db": "test",
        "collection": "fuzzing",
        "documents": [
            {
                "_id": 0,
                "number": 100,
                "str": "hello there",
                "obj":{"nested": "stuff"}
            }
        ]
    })
```
* An example request containing one kill cursor message:
    - This request contains a lego nested in a block, so that if this request is extended, future sulley primitives can reference the block by name.
```python
s_initialize("kill cursor")
if s_block_start("kill_cursor_msg"):
    s_lego("OP_KILL_CURSORS", options=
    {
        "requestID": 124098,
        "numberOfCursorIDs": 5,
        "cursorIDs": [
            2346245,
            123465663,
            76254,
            85662214,
            6245246
        ]
    })
s_block_end("kill_cursor_msg")
```
* An example of an update message
```python
s_initialize("update")
s_lego("OP_UPDATE", options=
    {
        "requestID": 56163,
        "db": "test",
        "collection": "fuzzing",
        "flags": 1,
        "selector": {
            "_id": 0,
        },
        "update": {
            "number": 11,
            "str": "Hello again",
            "obj":{"birds": "nest"}
        }
    })
```

## Developer info
### New Wire Messages
* Below is a rough template of what a new massage may look like when implemented using Sulley. Like all other MongoDB messages, it extends the [MongoMsg](./sulley/legos/MongoMsg.py) class.
```python
class OP_NEW(MongoMsg):
    """This sulley lego represents an OP_NEW MongoDB message"""
    def __init__(self, name, request, value, options):
        # Create the super class and push a header to the block.
        options = self.init_options(options, NEW_OPCODE)
        MongoMsg.__init__(self, name, request, options)
        
        # Save the appropriate options in case we need to reference them 
        # again in the future
        # set defaults
        self.db = options.get("db", "test")
        self.collection = options.get("collection", "fuzzing")
        self.flags = options.get("flags", NEW_FLAGS)
        self.document = options.get("document", {})
       
        # This command has 32 bits of reserved space.
        self.push(primitives.dword(0, signed=True))
        # cstring fullCollectionname
        self.push_namespace(self.db, self.collection)
        # int32 flags
        self.push(primitives.dword(self.flags, signed=True))
        # bson document
        self.push_bson_doc(self.document)

        # Always end with this command!
        self.end_block()
```
In order to reference this new lego in a request, you must also add the following line to [sulley/legos/\__init\__.py](sulley/legos/\__init\__.py)
```python
BIN["OP_NEW"] = mongo.OP_NEW
```

### Future development
* Switch command line parsing from getopt to the more pythonic [argparse](https://docs.python.org/dev/library/argparse.html)
* Sulley's **unix process monitor**
    - The process monitor's main function is to restart a mongod server after a crash occurs and store information about a crash.
    - This is currently under development and is not stable.
* **BSON document support**
    - The current handling of BSON documents is very simple and does not allow for much type specification.
