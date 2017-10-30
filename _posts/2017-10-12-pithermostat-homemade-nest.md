---
layout: post
title:  "PiThermostat - the Homemade Nest"
image: ''
date:   2017-10-12 00:06:31
tags:
- raspi
- automation
- programming
description: 'A simple tutorial for creating your own Nest-style thermostat using a RasPi, AWS, and more.'
categories:
- Projects 
---

PiThermostat is a project that I've been working on for quite some time. My goal was to create a cloud based thermostat solution for my apartment, which has a one-wire activation for the heater. I used NodeJS, which is run on Amazon AWS Lambda (connected by API Gateway, backed by DynamoDB) for the server component. The client component is based in Python, and is designed to work on any RasPi.

One of the coolest features of this app is the linking functionality, which allows you to give customers short TOTP codes to get API Key access to a particular device. This simplifies the process of allowing access to Tempurature on devices such as phones or TVs without having to manually type DeviceIDs and API Keys.

## What you need 
1. A Raspberry Pi, any version should work
2. An AWS Account (get a free tier account, very simple)
3. 1 <a target="_blank" href="https://www.amazon.com/gp/product/B01N9BA0O4/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B01N9BA0O4&linkCode=as2&tag=ianotto-20&linkId=996872bc013907032ad5e4f52645a4c3">DHT22/AMT2302 Temperature and Humidity Sensor</a><img src="//ir-na.amazon-adsystem.com/e/ir?t=ianotto-20&l=am2&o=1&a=B01N9BA0O4" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />
4. 1 <a target="_blank" href="http://amzn.to/2ii5g15">Relay Module for Arduino and Raspberry Pi 5V</a> (any should work, but make sure it's activation power is 5V)

**Note**: This tutorial is for thermostats that control ONLY heat. The software in the tutorial could easily be modified to control both, but it is not designed to work out of the box with it.

## Building the storage infrastructure
My first step was to setup the info storage structure on DynamoDB, since this would make it easier to test our Lambda functions in later steps.

**Note**: Table names, and column names are _case sensitive_ in this section. Please ensure that you enter them all exactly as written or the lambda functions may not work properly.

Login to your AWS console, and select a region that most closely matches your geographical area. At the top left, select "DynamoDB" under the Database heading. From here we will create our tables. Click on "Create Table" and create a table named "Devices", with a primary partition key of "deviceID". 

Then create a table with the name "LinkTable", with a primary partition key of "linkID". When created, we need to edit the TTL attribute of the table. To do this, select Tables on the left-side navigation bar, and click on LinkTable. It will show the properties of our table, one of which will be "TTL attribute". Simply click Manage TTL, and set the TTL Attribute to "creationTime".

## Building the Lambda functions
The next step was to, of course, write the functions that do the processing based on the info in the database. For simplicities sake, this section will only have the code snippets and names. For each code snippet, create a new Lambda function with the specified name. Also, depending on your region, you may have to update the section containing the AWS region for DynamoDB.

**userUpdateDeviceTemp**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.deviceID, event.body, callback);
    //callback(null, 'Hello from Lambda');
};

function make_call(deviceID, newConfig, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();

    var table = "Devices";
    newConfig = JSON.parse(newConfig);
    var params = {
        TableName:table,
        Key:{
            "deviceID": deviceID
        },
        UpdateExpression: "set info.current_temp = :s, info.current_humid = :h",
        ExpressionAttributeValues:{
            ":s": newConfig.current_temp,
            ":h": newConfig.current_humid
        },
        ReturnValues:"UPDATED_NEW"
    };

    docClient.update(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(null, {statusCode: 500, body: JSON.stringify({error: err})});
        } else {
            console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
            completion(null, {statusCode: 200, body: JSON.stringify({message: "success"})});
        }
    });
}

function parseOut(data, completion) {
    if(data.Count === 0) {
        completion(null, {statusCode: 404, body: JSON.stringify({error: "no such device ID", data: data})});
    } else {
        completion(null, {statusCode: 200, body: JSON.stringify(data.Items[0])});
    }
}
~~~~

**linkAddLink**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.linkID, event.body, callback);
    //callback(null, 'Hello from Lambda');
};

function generate_pseudorandom_link_code() {
    var bytes = require('crypto').randomBytes(4);
    return bytes.toString('hex').toUpperCase();
}

function make_call(uuid, apiKey, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();

    var table = "LinkTable";

    var params = {
        TableName: table,
        Item: {
            linkID: generate_pseudorandom_link_code(),
            creationTime: Math.floor(new Date() / 1000) + 60, //expire 60 seconds after creation
            info: {
                uuid: uuid,
                apiKey: apiKey
            }
        }
    };

    docClient.put(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(new Error(JSON.stringify(err)));
        } else {
            completion(null, {statusCode: 200, body: JSON.stringify({"linkID": params.Item.linkID})});
        }
    });
}
~~~~

**userUpdateDevice**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.deviceID, event.body, callback);
    //callback(null, 'Hello from Lambda');
};

function make_call(deviceID, newConfig, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();

    var table = "Devices";

    var params = {
        TableName:table,
        Key:{
            "deviceID": deviceID
        },
        UpdateExpression: "set info.settings = :s",
        ExpressionAttributeValues:{
            ":s": JSON.parse(newConfig)
        },
        ReturnValues:"UPDATED_NEW"
    };

    docClient.update(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(null, {statusCode: 500, body: JSON.stringify({error: err})});
        } else {
            console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
            completion(null, {statusCode: 200, body: JSON.stringify({message: "success"})});
        }
    });
}
~~~~

**userGetTemp**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.deviceID, callback);
    //callback(null, 'Hello from Lambda');
};

function make_call(deviceID, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();

    var table = "Devices";

    var params = {
        TableName : "Devices",
        KeyConditionExpression: "#did = :uuid",
        ExpressionAttributeNames:{
            "#did": "deviceID"
        },
        ExpressionAttributeValues: {
            ":uuid": deviceID
        }
    };

    docClient.query(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(new Error("A connection error occured."));
        } else {
            console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
            parseOut(data, completion);
        }
    });
}

function parseOut(data, completion) {
    if(data.Count === 0) {
        completion(null, {statusCode: 404, body: JSON.stringify({error: "no such device ID", data: data})});
    } else {
        completion(null, {statusCode: 200, body: JSON.stringify(data.Items[0].info)});
    }
}
~~~~

**userPutDevice**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.deviceID, event.body, callback);
    //callback(null, 'Hello from Lambda');
};

function make_call(deviceID, data, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();

    var table = "Devices";

    var params = {
        TableName: table,
        Item: {
            deviceID: deviceID,
            info: JSON.parse(data)
        }
    };

    docClient.put(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(new Error(JSON.stringify(err)));
        } else {
            completion(null, {statusCode: 200, body: JSON.stringify({message: "success"})});
        }
    });
}
~~~~

**linkGetLink**
~~~~
exports.handler = (event, context, callback) => {
    var call = make_call(event.pathParameters.linkID, callback);
    //callback(null, 'Hello from Lambda');
};

function make_call(linkID, completion) {
    var AWS = require("aws-sdk");

    AWS.config.update({
      region: "us-west-2"
    });

    var docClient = new AWS.DynamoDB.DocumentClient();


    var params = {
        TableName : "LinkTable",
        KeyConditionExpression: "#lid = :linkID",
        FilterExpression: ":currentTime < #expTime",
        ExpressionAttributeNames:{
            "#lid": "linkID",
            "#expTime": "creationTime"
        },
        ExpressionAttributeValues: {
            ":linkID": linkID,
            ":currentTime": Math.floor(new Date() / 1000)
        }
    };

    docClient.query(params, function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
            completion(new Error("A connection error occured."));
        } else {
            console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
            parseOut(data, completion);
        }
    });
}

function parseOut(data, completion) {
    if(data.Count === 0) {
        completion(null, {statusCode: 404, body: JSON.stringify({error: "no such link ID", data: data})});
    } else {
        completion(null, {statusCode: 200, body: JSON.stringify(data.Items[0].info)});
    }
}
~~~~

## Continue to Part 2
