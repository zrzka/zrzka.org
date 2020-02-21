---
title: AWS Lambda & API Gateway Error Handling
date: 2016-04-14T00:00:46+01:00
categories:
  - articles
tags:
  - aws
  - lambda
  - api-gateway
---

It’s fragile and kind of terrible. Let’s look at it. We have a function named _lambda_. It has one
argument named _action_ (enum with allowed values _data_, _exception_ or _timeout_). First thing we
would like to do is to validate input via [schema](https://github.com/plumatic/schema). Then we
would like to return custom error message (and status code) if it’s invalid or do other things
based on the _action_ argument value.

Here’s the code, just to demonstrate how it works.

```clj
(defn- timeout [ms]
  (let [c (chan)]
  (js/setTimeout (fn [] (close! c)) ms)
  c))

(defn- error
  [code message]
  (.stringify js/JSON (clj-&gt;js {:error-code code :error-message (pr-str message)})))

(def EventSchema
  {:action (s/enum "data" "exception" "timeout")})

(def ^:export lambda
  (async-lambda-fn
    (fn [event context]
      (go
        (if-let [e (s/check EventSchema event)]
          (fail! context (error "INVALID_PARAMS" e))
          (try
            (let [action (:action event)]
              (cond
                (= "data" action) (succeed! context {:some "data"})
                (= "timeout" action) (&lt;! (timeout 5000))
                (= "exception" action) (throw (js/Error. "Oops"))))
          (catch :default e
            (fail! context (error "EXCEPTION" e)))))))))
```

I’m not going to describe how to deploy Lambda function or API Gateway basics. I assume you know how
to do it. I’m going to focus on error handling only.

Our function is deployed, API Gateway resource (_/lambda_) and _GET_ method created and deployed as
well. All with default settings, just for now.

What happens when we try to use it?

```sh
curl -sw "n%{http_code}" -H "x-api-key: ..." -H "Accept: application/json"
https://...execute-api.eu-west-1.amazonaws.com/latest/lambda
```

```json
{
  "errorMessage": "{\"error-code\":\"INVALID_PARAMS\",\"error-message\":\"{:action missing-required-key}\"}"
}
200
```

JSON in JSON and status code is 200. **That’s not what we want. At all.**

## Pass action from query to lambda function

_POST_ method handles this automatically. Body is passed via _event_ argument. But _GET_ is a slightly different beast.

![Lambda GET method request](/images/aws/lambda-get-method-request.png)

We have to add _action_ to the URL Query String Parameters section. Then we have to add Body Mapping Template
(BMT) in the Integration Request section.

```json
{
  "action": "$input.params('action')"
}
```

Redeploy API and test.

```json
{
  "errorMessage": "{\"error-code\":\"INVALID_PARAMS\",\"error-message\":\"{:action (not (#{\"data\" \"exception\" \"timeout\"} \"\"))}\"}"
}
200
```

Much better, we can see validation response. Let’s try some [in]valid options. Status code 200 for all
of them with following responses.

### action=data

```json
{
  "some": "data"
}
```

### action=timeout

```json
{
  "errorMessage": "2016–04–14T12:20:20.584Z 3f0e0ca9–023b-11e6-a561-e58695fc8058 Task timed out after 4.00 seconds"
}
```

### action=exception

```json
{
  "errorMessage": "{\"error-code\":\"EXCEPTION\",\"error-message\":\"#object[Error Error: Oops]\"}"
}
```

NOTE: Our lambda function is deployed with _timeout_ set to 4 seconds. Just to make it working,
check the code, we’re waiting 5 seconds.

This is not good. We want 200 for success, 400 for invalid input parameters and 500 if something
bad happens (exception, timeout, …) and better structure of our response.

## Mapping status codes

We have to add 400 and 500 in the Method Response section. Otherwise we’ll not be able to select
these status codes in the Integration Response section.

![Lambda GET method response](/images/aws/lambda-get-method-response.png)

If lambda function fails, it returns JSON in the following format:

```json
{
  "errorMessage" : "whatever"
}
```

And the _errorMessage_ is a string. Do you want to return more than just a string? You have to encode it.

Open the Integration Response section and you’ll see something like this:

![Lambda GET method response status](/images/aws/lambda-method-response-status.png)

Default mapping uses 200 for the status code, there’s no regex and Body Mapping Template
(click on the triangle) is empty = passthrough. And because we know what our Lambda function
returns, we can add some regular expressions there.

![Lambda GET method response statuses](/images/aws/lambda-method-response-statuses.png)

What’s this all about? Lambda Error Regex column contains regular expressions that are matched against
_errorMessage_ from the lambda response if it indicates it failed (_fail!</em> in the code). For
_INVALID_PARAMS_ we would like to return 400, for any other error message (_.+_) we would like to
return 500 and 200 for the rest.

Redeploy and test.

### action=exception

```json
{
  "errorMessage": "{\"error-code\":\"EXCEPTION\",\"error-message\":\"#object[Error Error: Oops]\"}"
}
500
```

### action=timeout

```json
{
  "errorMessage": "2016–04–14T12:41:09.348Z 27638414–023e-11e6-bcf3–89b1797fdc7b Task timed out after 4.00 seconds"
}
500
```

### action=hallo

```json
{
  "errorMessage": "{\"error-code\":\"INVALID_PARAMS\",\"error-message\":\"{:action (not (#{\"data\" \"exception\" \"timeout\"} \"hallo\"))}\"}"
}
400
```

### action=data

```json
{
  "some": "data"
}
200
```

## Better response structure

Status codes solved. Now we have to focus on the response structure. It’s quite simple. We can do it
in the Integration Response section as well. We have to add Body Mapping Template for each regular
expression we already added.

![Lambda response body mapping](/images/aws/lambda-response-body-mapping.png)

![Code for the lambda response body mapping](/images/aws/lambda-response-body-mapping-code.png)

As you can see, BMT for 500 is more fancier that the one for 400. Why? AWS can kill our lambda function
because of timeout for example and then there’s just plain string in the _errorMessage_. Also our
lambda can catch exception, send back custom error map and then there’s encoded JSON in the
_errorMessage_. And because we **can’t add** two (or more) regular expressions for the same status
(500 in this case), we have to handle it in the BMT.

Redeploy and test.

### action=exception

```json
{
  "error-code": "EXCEPTION",
  "error-message": "#object[Error Error: Oops]"
}
500
```

### action=timeout

```json
{
  "error-code": "AWS",
  "error-message": "2016–04–14T13:50:09.212Z caf1daed-0247–11e6–91f8-d9f9a69fef2c Task timed out after 4.00 seconds"
}
500
```

### action=hallo

```json
{
  "error-code": "INVALID_PARAMS",
  "error-message" : "{:action (not (#{\"data\" \"exception\" \"timeout\"} \"hallo\"))}"
}
400
```

### action=data

```json
{
  "some":"data"
}
200
```

Goal reached. Status codes and better response structure for errors. It’s not perfect, there’s
always room for improvement, but it works.

It’s a bare minimum you have to do. Pretty rough, okay, terrible, but when you know how to do it …
Hope it helps, because I spent quite huge amount of time on this.
