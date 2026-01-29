# test-repo
Test 

[![Link Checker](https://github.com/erikkai/test-repo/actions/workflows/link-checker.yml/badge.svg)](https://github.com/erikkai/test-repo/actions/workflows/link-checker.yml)

> **NOTE:** This is a legacy code sample showing TeleSign customers how to set up a JWT service for use with the product App Verify. Links that are dead with no redirect are removed. Some of the products described here are no longer available. This project was created to help customers see the basics for what's involved to create a JWT service.

# Implement a JWT Service
This tutorial provides basic instructions for implementing a JWT service. You must have a server and JWT service in place if you want to use App Verify.

You use the JWT service to fetch a token for authentication. When presented to the TeleSign server, a check is performed to make sure the token contains the correct key. If it does, then whatever service you wanted to use for verification can proceed. So that you can perform a Get Status check, you store each token with an external ID (XID) in a database. The external ID is a unique number associated with a specific transaction. It enables you to retrieve details about the transaction from your database later. This tutorial stores the token with the associated XID and phone number, offering you two ways to easily reference transactions.

There are many ways you can implement a JWT service. This document is a guide to get you started, you may choose to do things a different way.

The steps for creating a JWT service are discussed in the following sections:

* [Requirements](#requirements)
* [How it Works](#how-it-works)
* [Implementation Walkthrough](#implementation-walkthrough)
* [Resources](#resources) 

# Requirements

For this tutorial, the following is required: 

* Your JWT server needs to use Transport Layer Security 1.1 or 1.2 if you plan to use TeleSign’s SDKs, as the TeleSign verification server uses these.
* App Verify Android JWT Service - this is available in the App Verify GitHub repository. The App Verify product is being discontinued. If you want to see this service, contact Support for access.
* MongoDB for Python or node-js

NOTE: This is not the only way to implement a JWT service. You can set it up many ways, using different products. You do not have to use MongoDB, for example, but the product name is listed as it was used to create the sample in this walkthrough.

If you want to follow along exactly with the tutorial, the following is used:

* Ubuntu version of Linux OS
* Python and Flask
* MongoDB

# How it Works 

This section provides a brief overview of the steps you will walkthrough in detail in the Implementation Walkthrough section.

1. On a server, set up a database using MongoDB.
2. Choose the language you want to implement your JWT service with. This tutorial provides instructions using Python.
3. Start MongoDB.
4. Create a JWT service module for your code. The module will contain, by the end of the tutorial:
    * A way to talk to MongoDB
    * Instructions for handling JWT tokens and the structure of the JWT tokens
    * Storage of the token with xid and phone number in a database
5. Start your service.

# Implementation Walkthrough 

1. Make sure you have your database set up and running. This tutorial uses MongoDB and associated commands for interacting with a MongoDB database. You can swap it out with another database product if you choose.
2. Import the services you need for your JWT service implementation. Not all the services may be installed on your system, you can easily add them with something like pip.

```
from json import dumps
from time import time
from datetime import datetime
from base64 import b64decode
from base64 import b64encode
from uuid import uuid4

import jwt

#MongoDB (replace with imports for your selected database software as needed)
import pymongo 

from flask import Flask
from flask import Response
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address 
```
NOTE: Be careful with the import jwt command. There is a second jwt folder, but import jwt is to add the PyJWT module. Make sure you have PyJWT and not just a folder for jwt, or some of the methods will not work.

3. Create a client connection to your MongoDB. (Or other database you choose, as necessary.)

```
client = pymongo.MongoClient()
```
4. Create a handle to your database. In this tutorial, the database you created to store tokens with their XID and phone number is called jwt_db.

```
jwt_db = client.jwt_db
```
5. Create a Flask instance. 

```
app = Flask(__name__)
```
6. Set up a rate limiter. This is a sample rate limiter, using a built-in feature of Flask. The limit is set very low for the walkthrough. You can choose a more appropriate setting from options listed in Flask’s documentation for [flask_limiter](https://flask-limiter.readthedocs.io/en/stable/). This will limit the number of requests that can be made for a given time.

7. Set up the route you want to retrieve tokens from.

```
@app.route('/v1/token/<phone_number>', methods=['GET'])
@app.route('v1/token/<phone_number>/<xid>', methods=['GET'])
```

8. Create the code for constructing and retrieving tokens and then storing them in your database. This code will take a phone number and an optional external ID (XID) as an argument. If you do not provide an XID as an argument, one is generated for you. XIDs are used to reference a specific transaction, for use with Get Status or Verification Callback.

NOTE: You must base64 decode your key before adding it with the rest of the token.

```
def get_token(phone_number, xid = None):
  response = Response(status=200, content_type='text/plain')
  response.headers['Allow'] = 'GET' 
  
  try:
    # Set token claims
    # Define the token time stamp validity range in seconds 
    iss = 'your_telesign_customer_id'
    iat = int(time()) - 10
    exp = iat + 90
    
    # Unique identifier to verify status of transaction completion from
    # TeleSign's GET status by XID service
    
    xid = xid or str(uuid4())
    
    payload = {
        'iss' : iss,
        'iat' : iat,
        'exp' : exp,
        'phn' : phone_number,
        'xid' : xid
    }
    
    # Base64 decode your API key 
    key = b64decode("your secret API key")
    
    # Base64 encode everything
    token = jwt.encode(payload, key)
    
    # Store the token in your DB (for use with MongoDB)
    result = jwt_db.tokens.insert_one(
        {
            'xid' : xid,
            'phone_number' : phone_number,
            'token' : token
        }
    ) 
    
  except Exception: 
      # Add logging as required here
      token = ""
      response.status = 500 
    
  return token

#FOR DEBUGGING (You can go to `0.0.0.0:5000/v1/token/<phone_number>/<xid>`)
if __name__ == "__main__": app.run(host = '0.0.0.0')
```

# Resources
This section provides a list of third party resources that will help you build your own JWT service:

* [www.jwt.io](https://www.jwt.io/) - jwt.io features a great debugger page. You can enter your encoded token and secret key, and the debugger will let you know if your token is formed properly.
* The Mongo DB Manual - A great manual for installing and configuring a MongoDB database.
* Getting Started with MongoDB (MongoDB Shell Edition) - Choose from Python, Node.JS, C++, Java, and C# editions.
* [Welcome to Flask](http://flask.palletsprojects.com/en/stable/) - Everything you need to know about Flask.
* [Flask-Limiter](https://pypi.org/project/Flask-Limiter/) - Details about Flask-Limiter, which provides rate limiting features to flask routes.
* [PyJWT](https://pypi.org/project/PyJWT/1.4.0/) - Information about PyJWT, which you use to encode tokens in this tutorial.


