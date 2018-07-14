# twiz (work in progress)

Twitter OAuth wizard.

Twiz does authentication and/or authorization to Twitter with OAuth 1.0a, has built in `REST` api support and also supports third party `STREAM` and `REST` libs.

## Intro
Many Twitter apis require user authentication (access token) before usage. OAuth 1.0a is (essencially a digital signature) process of letting know who (which app) wants to use an api and on which user's behalf. In other words you tell Twitter who you are, if twitter is ok with you it lets you to ask an user of your website (with twitter account), on authorization page, if he/she agrees that you act on its behalf (like post tweets on user's profile ect ...)

It happens to be a 3-leg (step) dance, it's implementation could look like this:
(------------------------- picture 1.0 - implementation example ---------------------)

As you can see there are *3* actors. Your web app/site, your server and twitter.  
Twitter apis for authorization and authentication do not use `CORS`, that is, they do not emit `CORS` headers. So we cant send request to twitter directly from browser, but rather proxy them trough a Server, since request sent from server do not have any CORS restrictions applied. Your app is identified with `CONSUMER_KEY` and `CONSUMER_SECRET`. Both of which you get from twitter when you [create new app](https://apps.twitter.com/).

On Twitter, user is internally associated with the access token.
 Like user's access token , your app's key and secret are extreamly sensitive information. If anyone whould to get your app's key/secret then it can send requests to twitter just like they were sent from your app. It can mock you. Usually that's not what you want. Also, javascript is in plain text form in browser so we should never have our `CONSUMER_KEY/SECRET` there but on server. This is another good reason why we need a server to proxy our requests. 

|Likewise if anyone is to get user's access token, then it may be able to send requests on user's behalf (who may actually never visited an app) and/or authorized such actions for that app. Altough, my guess is not in easy and straightforward way. 

	 
1. You ask your server to get you a request token. Server prepares and signs your request and sends it to twitter. Twitter checks your `CONSUMER_KEY` and signature (by reproducing it). If all is ok, it grants (returns) you an unauthorized *request_token*. It's function is to be authorized (approved) by user in step 2, so it can be used to get user's *access token* in step 3.

2. Uppon receiving request token data, user is redirected to twitter to obtain user authorization :
  * if you are using /oauth/authorize enpoint for redirection, then every time user is redirected there it lands on authorization [(interstitials) page](https://developer.twitter.com/en/docs/twitter-for-websites/log-in-with-twitter/guides/browser-sign-in-flow.html). Even if user previously authorized your app.
  
  * if you are using `/oauth/authenticate` enpoint for redirection, then only first time user is redirected it lands on authorization page. On any subsequent redirection twitter *remembers* first one and user is directed back again to the app. No authorization page is showed, the user is not involved directly. But historicaly it [didn't work](https://twittercommunity.com/t/twitter-app-permission-not-honored-on-subsequent-oauth-authenticate-calls/94440), like it should, for a time, 
 two were actually the same (authorization was just like authorize).

3. Since user approved your request token, now it is used to get user's access token. Server signs your request and sends it. Twitter does things similarly like in first step and grants you an access token which belongs to the user who authorized it in second step.

After we get user's access token the 3-leg OAuth is finished and we can use it to send request to twitter on user's behalf (like posts tweets on user's profile). In OAuth parlance this process of sending requests with the access token is called *accessing protected resources*, but it is not part of the OAuth.

We can see that in 3-rd leg access token is send back to web app. Which usually is not good idea, because of security implications it could have, like we mentioned earlier.
 
Let's see what twiz is doing with OAuth:
( -------------- picture Twiz basic ------------------- )

Three differences are:
 * **Optimized load** 
 
 Preparing an OAuth leg (step) mostly refers to the assembling of Signature Base String and Authorization Header string. It is all done in browser (Consumer) in an effort to ease the server load for actions that bear no security concerns and do not need to be executed in server. Already prepared requests come to the server who acts mostly like a signing/stamping authority by merely inserting sensitive data and signing requests.

* **Security**

Another important point is that the user's access token is never send back to the browser by any means.

* **Haste** 

     On the server we have a decision point where if we already have user's access token (stored from a previous authorization ie. in database) we can use haste (on diagram the *yes* branch ).
     
     Haste is a process where you verify access token freshness (verify credentials) with twitter and if it's fresh you can immediately go for a twitter api requests user actually wants. Checking token freshness is checking that user didn't revoke right to your app of doing things on it's behalf and such. 
     
     This is usefull for scenarios where you save user's access token after first authorization and then just chech for it's freshness before you go for an api request. User do not need to be bothered every time with 3-leg OAuth, there is no interstitials page. With haste you are the one who *remembers* user authorization instead of letting twitter to do it (like it does on the /oauth/authenticate). All in order to have smooth user experience, like for instance in case /oauth/authorize stops working as expected. 
     
     If this is the first time a user is making a request (and we dont have the access token) then we just continue the whole OAuth flow (on diagram the *no* branch). One of twiz's features is very easy switching between any of your oauth workflows while having a redundant machanism for smooth user experience (haste) as an option.

In order to efficiently and safely use twiz make sure you:
 1. **Provide HTTPS** all the way (client ---> server --> twitter),
 2. In browser install twiz-client, on server install twiz-server 
 3. Create [app account](https://apps.twitter.com/app/new) on twitter
 4. Users (of your app) must have twitter accounts 

## Usage 


in browser: 
    * CDN - <script src="https://cdn.jsdelivr.net/npm/twiz-client/client/twiz-client.min.js">
    * bower - comming soon
	 
on server:  
    * npm install twiz-server

### SPA (singe page apps)
*browser:*
```js  
 // Let's say this code is in your page ->  https://myApp.com 

let twizlent = twizClient();
  
btn.addListener('onClick', function(){                // lets say we initiate oauth on click event
  let args = {
      server_url:      'https://myServer.com/route', // address of your node server 
      redirection_url: 'https://myApp.com',          // address of your web app/site (where twitter will direct
                                                     //  user after authorization)
      options:{                                      //  twitter request options  
         method: 'POST',
         path:   'statuses/update.json'
         params: {
           status: "Hooray, new tweet!"
         }
      }
  }

  
  twizlent.OAuth(args)
  .then(function fulfilled(o){
      if(o.error)              // not 200OK responses (has o.error.statusCode, o.error.statusText, o.error.data)
      if(o.data)               // (200OK) will have data on succesfull twiz.haste(accessToken) call on server
      if(o.redirection)        // Will have an o.redirection set to *true* when twiz.continueOAuth() is called on                                     // server and user is redirected. Serves as a notifier for redirections.
      o.xhr                    // Always present in case you need to pull some data from response 
                               // (like custom server headers you might be sending)  
  }, function rejected(err){ // twiz errors
     // err is instance of Error()
     // has err.name, err.message, err.stack ...
  })

})  

// finishOAuth() Can be called asap in page 
// Makes 3-rd step from diagram 
// We dont need the redirection url for this step, but it will be ignored so we can pass same args
// It will fire after twitter (re)directs back to app, only on valid redirection (authorization) urls from twitter. 

twizlent.finishOAuth(args); 
  .then(function fulfilled(o){
      if(o.error) //  not 200OK responses
      if(o.data)  //  (200OK) will have data on succesfull twiz.continueOAuth() call on server
  
      o.xhr       // Always present in case you need to pull some data from response 
                  // (like custom server headers you might be sending)  
   }, function rejected(err){  // twiz errors
        // err is instance of Error()
        // has err.name, err.message, err.stack ...
}) 
```

Notice that our redirection_url is same as url of the page from which we are making a request. Making this a SPA use case.
The only presumtions about a succesfull request is one with 200OK status code, so anything that does not have that status code will still be in fulfilled handler but in o.error, left to your workflow judgement.

twizlent.OAuth() will bring api data (o.data) if *twiz.haste(accessToken)* was called on the server and had 200OK response. If not and the twiz.continueOAuth() is called it will receive request token and redirect user to twitter. 

Then o.redirection is set to true in fullfuled handler. Also note that here everything (redirection to twitter, twitter's (re)direction back to app) happens in same window/tab in browser. Check web site workflow for popUps[link].

### Authorize or Authenticate
By default twizlent.OAuth(..) will use the /oauth/authorize endpoint , but you can use the /oauth/authenticate like this:
```js
let args = {
    ...
      endpoints:{ 
         authorize: 'authenticate' // sets authenticate instead of authorize (notice no forward slash)
      }
 }
 ```


This is the so called [Sign in with Twitter](https://developer.twitter.com/en/docs/twitter-for-websites/log-in-with-twitter/guides/browser-sign-in-flow) flow, the one that uses /oauth/authenticate endpoint. That's how you would utilize it.

Server is writen as express middleware.
*node.js:*
```js
  var twizServer = require('twiz-server');
  var express    = require('express');
  
  var app = express();
  var twizer = twizServer({                             
         consumer_secret: process.env.CONSUMER_SECRET,  
         consumer_key:    process.env.CONSUMER_KEY,
         key:  fs.readFileSync('yourServerPrivateKey.pem'), 
         cert: fs.readFileSync('yourServerCert.pem')       // can be self signed certificate
  })

  app.use(twizer);                                          // use the twiz-server

  app.on('hasteOrOAuth', function(twiz, verifyCredentials){ // event where we pick haste or oauth
   
       // When you don't have access token (or just don't want to use haste) you continue the oauth flow
       twiz.continueOAuth(); 
                              // 1. user gets request token
                              // 2. is redirected for authorization (or authentication), twizlent.OAuth(..) has
                              //    o.redirection set to *true*
                              // 3. with twiz.finishOAuth() in browser users gets api data in o.data
       / *    . . .    */

       // Note that here in *hasteOrOAuth* handler is where you should go for user's access token
       // since 'hasteOrOAuth' event will only be emitted for certain requests. Otherwise you'll hog your server
       // cpu/io unnecessary. Verifyng credentials and using haste is completely optional step.

       verifyCredentials(accessToken,{ skip_status: true}) // When you have accessToken
       .then(function fullfilled(credentials){             // You can inspect returned credentials object
          twiz.haste(accessToken)                          // Gets api data and sends back to browser 
                                                           // (to twiz.OAuth(..) fullfiled handler)
       }, function rejected(err){ // non 200OK responses from verifyCredentials
            twiz.continueOAuth()  // likely you would want to send it to reauthorization of access token 
       })
       .catch(function(err){      // errors that might happen in fullfiled handler
     
       })
  })

  app.on('tokenFound', function(found){ // when whole oauth process is finished you will get the user's
                                        // access token 

     found                        // promise
     .then(function(accessToken){
         // user's access token received from twitter which you can put in database
         
     }, function rejected(err){   // twiz errors

     })
  })
]
```
### Access Token
   Currently the minimum of what twiz see as valid access token is an object that has properties *oauth_token* and *oauth_token_secret* set. But it can have other parameters, like *screen_name*.
The twiz-server (here twizer) is by default an ending middleware, that is it will end the request. So call it before your error handling middlewares, if any. There are cases when twiz does not end the request, check Stream. Errors will be sent to the next error handling midleware with *next(err)* calls and same  errors will also be piped back to the browser.

### Prefligh 
 If your app is not on same domain your browser will preflight request because of CORS. So you need to use some preflight middleware before twiz-server:
```js
 ...
 app.use(yourPreflight);
 app.use(twizer);
```
Currently you only have to set 'Access-Control-Allow-Origin' to your app's fqdn address. 
 
### Verify credentials 
 The credentials object in fulfileld handler can contain a lot of information. In order to ease the memory 
footprint you can use parameters object (like one with skip_status) to leave out information you don't need. Here [list of params](https://developer.twitter.com/en/docs/accounts-and-users/manage-account-settings/api-reference/get-account-verify_credentials.html) you can use.
 
////////////////////////////////////////////////////////////////////////////////////

### Web Site

Web Site workflow is very similar to that of a SPA. You just need to put the new_window object to args to specifiy your new popUp / window characteristics and call twizlent.finishOAuth(..)  from code in that popUp / window . Note that browser doesn't differentiate much between a popUp and a new window (new tab). Main difference is in dimentions.  

*browser:*
```js
 // Let's say this code is in your page ->  https://myApp.com 

let twizlent = twizClient();
  
btn.addListener('onClick', function(){                  // lets say we initiate oauth on click event
   let args = {
      server_url:      'https://myServer.com/route',    // address of your node server 
      redirection_url: 'https://myApp.com/popUpWindow', // address of your popUp/window page
                                                     
      new_window:{
         name: 'myPopUpWindow',
         features: 'resizable=yes,height=613,width=400,left=400,top=300'
      },

      options:{                                         //  twitter request options  
         method: 'POST',
         path:   'statuses/update.json'
         params: {
           status: "Hooray, new tweet!"
         }
      }
   }

   twizlent.OAuth(args)
   .then(function fulfilled(o){
      if(o.error)              // not 200OK responses (has o.error.statusCode, o.error.statusText, o.error.data)
      if(o.data)               // (200OK) will have data on succesfull twiz.haste(accessToken) call on server
      if(o.window)             // When redirection happens instead of o.redirection notification you'le have 
                               // reference to the popUp/window and the redirection will happen from that window.                               // Like you would expect. 

       o.xhr                   // Always present in case you need to pull some data from response 
                               // (like custom server headers you might be sending)  
   }, function rejected(err){  // Twiz errors
        // err is instance of Error()
        // has err.name, err.message, err.stack ...
   })

})
```
The redirection_url is now different then the page url then one from which we are making the request. Also we have new_window where we specify the window/popUp features where redirection url will land . Making this more of a website use case.
The new_window object contains two properties, name and features, they act the same as windowName and windowFeatures in [window.open()](https://developer.mozilla.org/en-US/docs/Web/API/Window/open). Note o.window reference to newly opened window / popUp instead of o.redirection. 

*browser(different page):*
```js
 // code in https://myApp.com/popUpWindow
  twizlent.finishOAuth(args);  // Also can be called asap in page
  .then(function fulfilled(o){
      if(o.error)              //  not 200OK responses
      if(o.data)               //  (200OK)  will have data on succesfull twiz.continueOAuth() call on server

      o.xhr                     // always present in case you need to pull some data from response 
                               // (like custom server headers you might be sending)      
   }, function rejected(err){  // twiz errors
        // err is instance of Error()
        // has err.name, err.message, err.stack ...
   })
]
```
What this enables is to have completely custom popUp pages but same familiar popUp like for instance when you whould like to share something on twitter by pressing a twitter share button. Currently the downside is that users of the web site use case will get a popUp warning by browser which they have to allow before popUp apears.
Test drive [here]
                             /////         ADDITIONAL USAGE        /////////
*node.js:*
```js 
  // Same code as in SPA use case;
```
### getSessionData 

There is an interesting capability provided by the OAuth 1.0a spec section 6.2.3. "The callback URL MAY include Consumer provided query parameters. The Service Provider MUST retain them unmodified and append the OAuth parameters to the existing query".
 This relates to OAuth step 2. When we redirect user to twitter for obtaining authorization we are *sending* a callback url (I've called it redirection_url) along with request token (not shown in diagrams), which twitter uses to (re)direct user back to app when authorization is done. In that url we can piggy back arbitrary data as query params (to twitter and back to app). Then, when we are (re)directed back to app, we can take back that data. The result is that we have a simple mechanism that allows our data to survive redirections, that is changing window contexts in browser. Which is handy in cases when we have the SPA workflow and everthing happens in one window tab, so data we send from our app's window context can *survive* changing that context to the context of twitter's window on which oauthorization happens and then again finally our apps' window context.

 This can also be used for web site workflows, but you'le get the *o.window* reference in that case which also can be used for exact same thing. This mechanism comes in hand when you are in a place like github pages and don't have access to a database there and/or your are not currently interested in puting a database solution on a server. Here is how you can use it.

#### SPA
*browser:*
```js
 //  code in https://myApp.com
  let twizlent = twizClient();
  
  btn.addListener('onClick', function(){                  // lets say we initiate oauth on click event
     let args = {
        server_url:      'https://myServer.com/route',    // address of your node server where twiz-server runs
        redirection_url: 'https://myApp.com/popUpWindow', // address of your popUp/window page
         
        session_data: {                                  // our  arbitrary session data we want 
           weater: 'Dry, partly cloudy with breeze.',
           background_noise: 'cicada low frequency',
           intentions: 'lawfull good'
        }                                            
        new_window:{
           name: 'myPopUpWindow',
           features: 'resizable=yes,height=613,width=400,left=400,top=300'
        },

        options:{                                         //  twitter request options  
           method: 'POST',
           path:   'statuses/update.json'
           params: {
             status: "Hooray, new tweet!"
           }
       }
     }
  }

```
 *browser(different page):*
 ```js
  // https://myApp.com/popUpWindow
  let twizlent = twizClient();

  let sessionData = twizlent.getSessionData();   // Gets our session_data from re(direction) url 
                                                 // Can also be called asap in page 
  ...     
]
```
| onEnd |

 There is a second argument that is passed to your 'tokenFound' handler. The onEnd(..) function.
It's use is to specify your function that will end the request as you see fit. For instance when you would like to use a template engine. onEnd(..) fires afther access protected resources (api) call but it does not end the response.


SERVER CODE >>>>> [
  app.on('tokenFound', function(found, onEnd){

     found                        
     .then(function(accessToken){
         // user's access token received from twitter which you can put in database
         
     }, function rejected(err){   // twiz errors

     })

     onEnd(function(apiData, res){ // happens after accessToken is found
         res.render('signedInUI', { user: apiData.user.name })   // Uses server side template engine
                                                                // Ends response internally
     })
  })
]

When we get the accessToken in our promise then twiz gets api data and calls your onEnd callback with that data and response stream.
So we've sent the rendered html with user.name from data we got from  twitter's statuses/update.json
api. When you are specifying the onEnd function then it must end the response or else the request will hang. 

Also if your workflow requires front-end template rendering. You can instead on res.render use :
>>> res.redirect(302,'/signedInUI');  // redirects the client to the signedInUI template

Then the twizlent.finishOAuth(..) will get this 'signedInUI' template in it's 'o.data'.

| beforeSend |

On client side the args.options object can alse have a 'beforeSend' property. It is a function that allows your to manipulate xhr instance before request is sent.
browser code >>>>
[   
// https://myApp.com
  btn.addListener('onClick', function(){                  // lets say we initiate oauth on click event
     let args = {
        ...
        ...
        options:{                                         //  twitter request options  
           method: 'POST',
           path:   'statuses/update.json'
           params: {
             status: "Hooray, new tweet!"
           },
           beforeSend: function(xhr){
              // xhr.open(..) is called for you, dont do it
       
                 xhr.setRequestHeader('X-My-Header-Name', 'yValue') // in case you whould ever need something like this

              // xhr.send(..) is called for you, dont do it
           }
       }
     } 

]


| Callback |
The args object can have the 'callback' property. You can specify there your callback function which will run if two cases are met:

1. Promise is not avalable
2. You've set the callback property

args = {
   ...
   ...
   callback: function(o){
         if(o.error)
         if(o.data)
         if(o.window) // or if(o.rederection). When using SPA workflow (o.window), 
                      // when using web site (o.redirection)  

         o.xhr        // always present in case you need to pull some data from response (like custom headers)      
   }
   
}

try{
   let twizlent = twizClient();
   twizlent.OAuth(args)
}
catch(err){
   // twiz errors
}


If promise is not avalale and there is no callback specified you'le get and error 'noCallbackFunc' see errors[link]

| Stream |

With twiz using stream means using third party STREAM and/or REST libraries, while letting twiz to take care only for user authentication, that is getting an access token. In other words stream efectively turns off built in REST capability.
Specify stream in client by passing 'args.stream = true'. 

BROWSER [
  ...
  let twizlent = twizClient();

  let args = {
     server_url: 'myServer.com/route',
     ...
     ...
     stream: true , // Indicate that you want to use 'your own' Stream and/or REST libs 
     options : {              // twitter request options
        path:   'media/upload',
        method: 'POST',
        params: {
           source: 'image.jpg',
           ...  // can contain your own properties 
           ... 
        }
     }
  }
 
  twizlent.OAuth(args)
  .then(..)          
     
  ...
  ...
  twizlent.finishOAuth(args)
  .then(..)
 
]

Then on server you can do the following:

SERVER
[
   app.use(twizer)                                           // instance of twiz-server

   app.on('hasteOrOAuth', function(twiz, verifyCredentials){
                                  // Here we presume access token is already loaded from your storage
       if(!accessToken){
         twiz.continueOAuth();    // we continue OAuth when we don't have access token
         return;
       }
       
       // If you are storing tokens in persistent memory code can look like this: 
       // (this step is optional)
 
       verifyCredentials(accessToken, {skip_})              
       .then(function fullfiled(credentials){
            if(twiz.stream){                          // Check that user indicated stream
               app.options     = twiz.twitterOptions; // Twitter request options as in the args.options on client
               app.accessToken = accessToken;         // save access token to app (just as an example)     
               twiz.next()                            // Jump to next middleware, here it's 'myStream'                                 
            }
            else twiz.haste(accessToken);            // maybe you'll want haste
       }, function rejected(err){
            // ...
       })
   })

   app.on('tokenFound', function(found, twiz){
       
       found
       .then(function fullfiled(accessToken){
         
           if(twiz.stream){
              app.options     = twiz.twitterOptions; // Save twitter request options
              app.accessToken = accessToken;         // Save access token
              twiz.next()                            // Jump to next middleware
           }            

       }, function rejected(err){
         ...
       }) 
      
   })
   app.use(function myStream(req, res, next){      // your own streaming implementation, must end response 
       let accessToken = app.accessToken
       let options     = app.twitterOptions            // same as in args.options

      // your code for STREAM and/or REST apis ...
   })
]

Instead of baking in something like onStream(..) function that would handle your own 'stream/rest'requests, in 'hasteOrOAuth' and 'tokenFound' events you are given 3 basic building blocks for creating such 'onStream' handler. With the exception of accessToken that you've got already, they are:
  twiz.stream         // flag that indicates request wants third party stream/rest capability
  twiz.twitterOptions // your args.options from browser, has 'path', 'method' and 'params' properties
  twiz.next           // reference to Express' next() function which runs next middleware (myStream in example)

So you can easily see something like onStream handler that:
 
 1. checks if request wants custom stream/rest libs
 2. saves twitterOptions/accessToken to apropriate places to be used in the next middleware 
 3. when it's done doing its thing, calls the next middleware

As you can see, twiz is not keen to stuff potentialy security sensitive data to 'app' or 'req' objects. It is given to your judgement to place your data where you see fit. 'app' is used as storage in example just as an ilustration of a purpose, maybe you whould want some caching solution like redis/memcache. 


| Chunked responses| 
When making stream requests the response often come as series of data chunks[link] from other end. To consume response in chunk by chunk manner set xhr.onprogress(..)[link] callback in beforeSend function:
>>>>> [
  let args = {
      ...
      options:{
         ...
         beforeSend: function(xhr){

            xhr.onprogress = function(evt){
              xhr.responseText // consume chunks (for text/plain response, for instance)
             }
         }
      }
  }
A reference[link] on how to consume chunks. 
1. If your not sending  'content-type'.
      It is good idea to set 'content-type' header on your server before you proxy first chunk back to client or else when stream finialy ends promise will reject with 'noContentType' error. But your will be already consumed in your in onprogress(..) callback.

2. If you are sending content-type.
When your stream is consumed in onprogress() and it ends the promise will still resolve and you will have all your data that stream emmited in o.data. Since your getting your data in onprogress(..) you might not want to receive it in your promise too. Same goes if your using callbacks and not promises. To stop all data from stream to resolve in promise set 'chunked=true' in args.options.
 let args = {
    ...
    options = {
       ...
       chunked: true
    }
 }
  
]

By setting 'chunked' you dont have to worry about sending content-type, it will make the promise reject with error 'chunkedResponseWarning' no matter the presents of content-type header. So you have consistent behavior, when you would want to consume chunks only in xhr.onprogress(..) handler.   


| Errors |
<Client>
twizlent.OAuth(..) 'rejected()' handler:
 <error.name>     <error.message>

redirectionUrlNotSet: "You must provide a redirection_url to which users will be redirected.",
serverUrlNotSet:  "You must proivide server_url to which request will be sent",
optionNotSet:     "Check that 'method' and 'path' are set."

noCallbackFunc: 'You must specify a callback function',
callbackURLnotConfirmed: "Redirection(callback) url you specified wasn't confirmed by Twitter"
noContentType: "Failed to get content-type header from response. Possible CORS restrictions or header is missing."
 chunkedResponseWarning: 'Stream is consumed chunk by chunk in xhr.onprogress(..) callback



twizlent.finishOAuth(..) 'rejected' handler:

 verifierNotFound: '"oauth_verifier" string was not found in redirection(callback) url.',
 tokenNotFound: '"oauth_token" string was not found in redirection(callback) url.',
 
 tokenMissmatch: 'Request token and token from redirection(callback) url do not match',
 This error can happen if user  

 requestTokenNotSet: 'Request token was not set',
 requestTokenNotSaved: 'Request token was not saved. Check that page url from which you make request match your redirection_url.', 
chunkedResponseWarning: 'Stream is consumed chunk by chunk in xhr.onprogress(..) callback
 noRepeat: "Cannot make another request with same redirection(callback) url",
 // urlNotFound: "Current window location (url) not found",
// as console warn in SessionData = noSessionData: 'Unable to find session data in current url',
 spaWarning: 'Twitter authorization data not found in url'.

'spaWarning' and 'noRepeat' are errors that have informative character and usually you dont have to pay attention to them. They happen when user loads/relods page where twizlent.finishOAuth(..) is called on every load, imediately (which is valid). They are indication that twizlent.finishOAuth(..) will not run. For example, 'spaWarning' means finishOAuth() won't run on url that doesn't contain valid twitter authorization data. 'noRepeat' means that you cannot make two requests with same twitter authorization data (like same request token). 

<Server>
twiz.continueOAuth() Errors are ones that can happen on request or response streams (low level) and they are hanled by calling next(..). There are no twiz errors currently for this function. Not 200OK responses are only piped back to client and are not considered as errors.

twiz.haste() errors work same as continueOAuth()

verifyCredentials() any not 200OK response are considered as an 'accessTokenNotVerified' error. Express' next(..) is called and promise is rejected with the same error. 
<error.name> <error.message>
'accessTokenNotVerified': '';

Note that the error.message will be a json string taken from response payload so you can have exact twitter error description, error code etc ...
  



