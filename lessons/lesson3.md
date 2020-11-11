## Lesson 3: Fire Event using Event SDK

### Fire Event
Once you set up the app and register the event provider, now you can make user click like button as fire event by adding code to your project firefly app, this lesson will walk through the code in publish-event, test it on the UI with "Invoke" button and see the success response

When you run below command 
```bash
aio app init
```
It provide you with the template of `publish event` to provide a way to convert an input json to cloud event and publish to events.
![event-provider](assets/publish-event-cli.png)

You can choose to use this template code at `/actions/publish-events/index.js` or create your own code.
Within the newly created app, Firstly, set up `package.json` with the lists of dependencies, version, etc. 
Then `manifest.yml` lists the declaration of serverless actions including name, source files, runtime kind, default params, annotations, and so on. In this lesson, we will choose to use this template to modify the code to our need.

Note: here put in the `providerId`,`apiKey` and `eventCode`from lesson 2 in the `manifest.yml` and `orgId`,`accessToken`can be passed through `headers`

Below is a sample `manifest.yml` 
```javascript
packages:
  __APP_PACKAGE__:
    license: Apache-2.0
    actions:
      <project-name>:
        function: actions/<project-name>/index.js
        web: 'yes'
        runtime: 'nodejs:12'
        inputs:
          LOG_LEVEL: debug
        annotations:
          require-adobe-auth: true
          final: true
      publish-events:
        function: actions/publish-events/index.js
        web: 'yes'
        runtime: 'nodejs:12'
        inputs:
          LOG_LEVEL: debug
          apiKey: <Your-SERVICE_API_KEY>
          providerId: <YOUR-PROVIDER_ID>
          eventCode: <YOUR-EVENT_CODE>
        annotations:
          require-adobe-auth: true
          final: true
```

Now let's start to take a deeper look the tempalte code: 

* Source code is at `actions/publish-events/index.js`
* It is a [web action](https://github.com/AdobeDocs/adobeio-runtime/blob/master/guides/creating_actions.md#invoking-web-actions)
* The action will be run in the `nodejs:12` [runtime container on I/O Runtime](https://github.com/AdobeDocs/adobeio-runtime/blob/master/reference/runtimes.md)
* It has some [default params](https://github.com/AdobeDocs/adobeio-runtime/blob/master/guides/creating_actions.md#working-with-parameters) such as `LOG_LEVEL`, you can pass in your `params` like `apiKey`, `provideId` and `eventCode`from `manifest.yml` 

Now let's have a deeper look at the action's source code from template:

```javascript
/*
* <license header>
*/

/**
 * This is a sample action showcasing how to create a cloud event and publish to I/O Events
 *
 * Note:
 * You might want to disable authentication and authorization checks against Adobe Identity Management System for a generic action. In that case:
 *   - Remove the require-adobe-auth annotation for this action in the manifest.yml of your application
 *   - Remove the Authorization header from the array passed in checkMissingRequestInputs
 *   - The two steps above imply that every client knowing the URL to this deployed action will be able to invoke it without any authentication and authorization checks against Adobe Identity Management System
 *   - Make sure to validate these changes against your security requirements before deploying the action
 */


const { Core, Events } = require('@adobe/aio-sdk')
const uuid = require('uuid')
const cloudEventV1 = require('cloudevents-sdk/v1')
const { errorResponse, getBearerToken, stringParameters, checkMissingRequestInputs } = require('../utils')

// main function that will be executed by Adobe I/O Runtime
async function main (params) {
  // create a Logger
  const logger = Core.Logger('main', { level: params.LOG_LEVEL || 'info' })

  try {
    // 'info' is the default level if not set
    logger.info('Calling the main action')

    // log parameters, only if params.LOG_LEVEL === 'debug'
    logger.debug(stringParameters(params))

    // check for missing request input parameters and headers
    const requiredParams = ['apiKey', 'providerId', 'eventCode', 'payload']
    const requiredHeaders = ['Authorization', 'x-gw-ims-org-id']
    const errorMessage = checkMissingRequestInputs(params, requiredParams, requiredHeaders)
    if (errorMessage) {
      // return and log client errors
      return errorResponse(400, errorMessage, logger)
    }

    // extract the user Bearer token from the Authorization header
    const token = getBearerToken(params)

    
    // initialize the client
    const orgId = params.__ow_headers['x-gw-ims-org-id']
    const eventsClient = await Events.init(orgId, params.apiKey, token)

    // Create cloud event for the given payload
    const cloudEvent = createCloudEvent(params.providerId, params.eventCode, params.payload)

    // Publish to I/O Events
    const published = await eventsClient.publishEvent(cloudEvent)
    let statusCode = 200
    if (published === 'OK') {
      logger.info('Published successfully to I/O Events')
    } else if (published === undefined) {
      logger.info('Published to I/O Events but there were not interested registrations')
      statusCode = 204
    }
    const response = {
      statusCode: statusCode,
    }

    // log the response status code
    logger.info(`${response.statusCode}: successful request`)
    return response
  } catch (error) {
    // log any server errors
    logger.error(error)
    // return with 500
    return errorResponse(500, 'server error', logger)
  }
}

function createCloudEvent(providerId, eventCode, payload) {
  let cloudevent = cloudEventV1.event()
    .data(payload)
    .source('urn:uuid:' + providerId)
    .type(eventCode)
    .id(uuid.v4())
  return cloudevent.format()
}
exports.main = main

```
What happens here is that the action exposes a `main` function, which accepts a list of params from the client. It checks the required params for using the `cloudevents-sdk`. 

Next, let's see how the web UI communicates with the backend. In `web-src/src/components` we already provide a template of UI.
After you select the Actions to `publish-events` and then click the `Invoke` button, it will invokes the action. In the action, it will send out the event. When you invoke, you could also add actual params, in this example, we add `{"payload": "you got a like"}`, in the webhook result, you will see the payload showed in `{"data": "you got a like"}`.
![templateui](assets/template-ui.png)
![eventresult](assets/event-webhook-result.png)


### Run and Deploy the Firefly App
You can run the firefly app locally by execute the below command with AIO CLI
```bash
aio app run
```
This command will deploy the `publish-event` action into I/O Runtime, and spins up a local instance for the UI. When the app is up and running, it can be seen at `https://localhost:9080`. You should be able to see the UI of the app and it is also possible to access the app from ExC Shell: `https://experience.adobe.com/?devMode=true#/apps/?localDevUrl=https://localhost:9080`. You might be asked to log in using your Adobe ID.  When the website is opened, the UI is almost similar to what you see when deployed on localhost, except the ExC Shell on top of the UI.

Once you are satisfied with your app, you can deploy your app by run below command:
```bash
aio app deploy
```
This command will deploy the app to your namespace, you will get the URL like 
`https://<Runtime-namespace>.adobeio-static.net/<project-name>-0.0.1/index.html`
Now if you modify the UI code to make click the `like` button to trigger the event, you should be able to fire the event 

Next lesson: [Lesson4](lesson4.md)