# Raygun4js

Forked version of Raygun is added to provide the sourcemap integration for mobile platforms.

It extends the source map functionality provided by Raygun for web version for all the frotend Hybrid apps e.g; Ionic.

## How it works.

I assuming you have fare idea of how Raygun plugin works. Else take a look at its documentation below or at https://github.com/MindscapeHQ/raygun4js.

Please read https://raygun.io/blog/2014/02/getting-started-with-javascript-source-maps/ for detailed description on how sourcemap works in Raygun.


All platforms put different url in the sourcemap that is uploaded to Raygun :

a. ANDROID : file:///android_asset/www/scripts/application-scripts.min.js
b. IOS : file:///private/var/mobile/Containers/Bundle/Application/{{uuid}}/Hybrid.app/www/scripts/application-scripts.min.js

And the device URL is published as : file:///android_asset/www/index.html

We face three major issues : 

1. In case of ios all device would have different url and hence finding sourcemap is litteraly becomes impossible.
2. Device url and sourcemap url is different hence mapping can not be done even in case of android. Notice Device url has <b>"index.html"</b>.
Hence we need to remove the index.html from Device URL if we need a proper mapping between sourcemap and stactrace.
3. Plugin to fecilitate this wthout any issue.

Hence to solve all problem I have updated the plugin to make sure sourcemap can be created without any issue :

I have added four new values in "options" parameter passed while initialization of Raygun.

1. domainName : These could be any domain name with which mobile app will send the stacktrace to raygun. 
As we need a global url to fetch the sourcemap for the stacktrace we need to replace the local URL of mobile platform with global URL.

2. platformType : [IOS/ANDROID/WEB] : As release to all the platforms can happen at different point of time, its a good idea to keep the sourcemap for all with different relative URI.

3. versionNumber : To map the sourcemap of different version with the stacktrace.

4. landingPage : index.html as described in example above. The landing page is replaced from the above device URL and hence device URL and stacktrace URL becomes same.

So final url to which the stacktrace is uploaded to Raygun is summation of all the 3 points mentioned above.

So suppose I initialized Raygun with following 
```options 
{"domainName":"https://raygun-mobile-sourcemap.com", "platformType":"IOS", "versionNumber":"v1.3", "landingPage":"index.html"}
```
Final Url Will be :
```URL
https://raygun-mobile-sourcemap.com/IOS/v1.3
```

Hence the stacktrace now uploaded to Raygun will try to resolve the sourcemap from the files uploaded at above URL in raygun.

## Usage

In your web page:

```html
<script src="dist/raygun.min.js"></script>
<script>
  Raygun.init('yourApiKey');
</script>
```

To submit manual errors:

```javascript
Raygun.init('yourApiKey');
try {
  // your code
  throw new Error('oops');
}
catch(e) {
  Raygun.send(e);
}
```

In order to get stack traces, you need to wrap your code in a try/catch block like above. Otherwise the error hits ```window.onerror``` handler and will only contain the error message, line number, and column number.

You also need to throw errors with ```throw new Error('foo')``` instead of ```throw 'foo'```.

To automatically catch and send unhandled errors, attach the window.onerror handler call:

```html
<script src="dist/raygun.min.js"></script>
<script>
  Raygun.init('yourApiKey').attach();
</script>
```

If you need to detach it (this will disable automatic unhandled error sending):

```javascript
Raygun.detach();
```

If you are serving your site over HTTP and want IE8 to be able to submit JavaScript errors then you will
need to set the following setting which will allow IE8 to submit the error over HTTP. Otherwise the provider
will only submit over HTTPS which IE8 will not allow while being served over HTTP.

```javascript
Raygun.init('yourApiKey', { allowInsecureSubmissions: true });
```

## Documentation

### Initialization Options

Pass in an object as the second parameter to init() containing one or more of these keys and a boolean to customize the behavior:

`allowInsecureSubmissions` - posts error payloads over HTTP. This allows **IE8** to send JS errors

`ignoreAjaxAbort` - User-aborted Ajax calls result in errors - if this option is true, these will not be sent.

`ignoreAjaxError` - Ajax requests that return error codes will not be sent as errors to Raygun if this options is true.

`debugMode` - Raygun4JS will log to the console when sending errors.

`wrapAsynchronousCallbacks` - if set to `false`, async callback functions triggered by setTimeout/setInterval will not be wrapped when attach() is called. _Defaults to true_

`ignore3rdPartyErrors` - ignores any errors that have no stack trace information. This will discard any errors that occur completely
within 3rd party scripts - if code loaded from the current domain called the 3rd party function, it will have at least one stack line
and will still be sent.

`excludedHostnames` - Prevents errors from being sent from certain hostnames (domains) by providing an array of strings or RegExp
objects (for partial matches). Each should match the hostname or TLD that you want to exclude. Note that protocols are not tested.

`excludedUserAgents` - Prevents errors from being sent from certain user agents by providing an array of strings. This is very helpful to exclude errors reported by certain browsers or test automation with `CasperJS`, `PhantomJS` or any other testing utility that sends a custom user agent. If a part of the client's `navigator.userAgent` matches one of the given strings in the array, then the client will be excluded from error reporting.

An example:

```javascript
Raygun.init('apikey', {
  allowInsecureSubmissions: true,
  ignoreAjaxAbort: true,
  ignoreAjaxError: true,
  debugMode: true,
  ignore3rdPartyErrors: false,
  excludedHostnames: ['localhost', '\.development']
  excludedUserAgents: ['CasperJS', 'Chrome/42', 'Android']
}).attach();
```

### Multiple Raygun objects on a single page

You can now have multiple Raygun objects in global scope. This lets you set them up with different API keys for instance, and allow you to send different errors to more than one application in the Raygun web app.

To create a new Raygun object and use it call:

```javascript
var secondRaygun = Raygun.constructNewRaygun();
secondRaygun.init('apikey');
secondRaygun.send(...)
```

Only one Raygun object can be attached as the window.onerror handler at one time, as *onerror* can only be bound to one function at once. Whichever Raygun object had `attach()` called on it last will handle the unhandle errors for the page.

### Callback Events

#### onBeforeSend

Call `Raygun.onBeforeSend()`, passing in a function which takes one parameter (see the example below). This callback function will be called immediately before the payload is sent. The one parameter it gets will be the payload that is about to be sent. Thus from your function you can inspect the payload and decide whether or not to send it.

From the supplied function, you should return either the payload (intact or mutated as per your needs), or `false`.

If your function returns a truthy object, Raygun4JS will attempt to send it as supplied. Thus, you can mutate it as per your needs - preferably only the values if you wish to filter out data that is not taken care of by `filterSensitiveData()`. You can also of course return it as supplied.

If, after inspecting the payload, you wish to discard it and abort the send to Raygun, simply return `false`.

By example:

```javascript
var myBeforeSend = function (payload) {
  console.log(payload); // Modify the payload here if necessary
  return payload; // Return false here to abort the send
}

Raygun.onBeforeSend(myBeforeSend);
```

### Sending custom data

**On initialization:**

Custom data variables (objects, arrays etc) can be added by calling the withCustomData function on the Raygun object:

```javascript
Raygun.init('{{your_api_key}}').attach().withCustomData({ foo: 'bar' });
```

They can also be passed in as the third parameter in the init() call, for instance:

```javascript
Raygun.init('{{your_api_key}}', null, { enviroment: 'production' }).attach();
```

**During a Send:**

You can also pass custom data with manual send calls, with the second parameter. This lets you add variables that are in scope or global when handled in catch blocks. For example:

```javascript
Raygun.send(err, [{customName: 'customData'}];
```

#### Providing custom data with a callback

To send the state of variables at the time an error occurs, you can pass withCustomData a callback function. This needs to return an object. By example:

```javascript
var desiredNum = 1;

function getMyData() {
 return { num: desiredNum };
}

Raygun.init('apikey').attach().withCustomData(getMyData);
```

`getMyData` will be called when Raygun4JS is about to send an error, which will construct the custom data. This will be merged with any custom data provided on a Raygun.send() call.

### Adding tags

The Raygun dashboard can also display tags for errors. These are arrays of strings or Numbers. This is done similar to the above custom data, like so:

**On initialization:**

```javascript
Raygun.init('{{your_api_key}}').attach().withTags(['tag1', 'tag2']);
```

**During a Send:**

Pass tags in as the third parameter:

```javascript
Raygun.send(err, null, ['tag']];
```

#### Adding tags with a callback function

As above for custom data, withTags() can now also accept a callback function. This will be called when the provider is about to send, to construct the tags. The function you pass to withTags() should return an array (ideally of strings/Numbers/Dates).

### Affected user tracking

By default, Raygun4JS assigns a unique anonymous ID for the current user. This is stored as a cookie. If the current user changes, to reset it and assign a new ID you can call:

```js
Raygun.resetAnonymousUser();
```

To disable anonymous user tracking, call `Raygun.init('apikey', { disableAnonymousUserTracking: true });`.

#### Rich user data

You can provide additional information about the currently logged in user to Raygun by calling:

```javascript
Raygun.setUser('unique_user_identifier');
```

This method takes additional parameters that are used when reporting over the affected users. the full method signature is

```javascript
setUser: function (user, isAnonymous, email, fullName, firstName, uuid)
```

`user` is the user identifier. This will be used to uniquely identify the user within Raygun. This is the only required parameter, but is only required if you are using user tracking.

`isAnonymous` is a bool indicating whether the user is anonymous or actually has a user account. Even if this is set to true, you should still give the user a unique identifier of some kind.

`email` is the user's email address.

`fullName` is the user's full name.

`firstName` is the user's first or preferred name.

`uuid` is the identifier of the device the app is running on. This could be used to correlate user accounts over multiple machines.

This will be transmitted with each message. A count of unique users will appear on the dashboard in the individual error view. If you provide an email address, the user's Gravatar will be displayed (if they have one). This method is optional; if it is not called no user tracking will be performed. Note that if the user context changes (such as in an SPA), you should call this method again to update it.

**Resetting the user:** you can now pass in empty strings (or false to `isAnonymous`) to reset the current user for login/logout scenarios.

### Version filtering

You can set a version for your app by calling:

```
Raygun.setVersion('1.0.0.0');
```

This will allow you to filter the errors in the dashboard by that version. You can also select only the latest version, to ignore errors that were triggered by ancient versions of your code. The parameter needs to be a string in the format x.x.x.x, where x is a positive integer.

### Filtering sensitive data

You can blacklist keys to prevent their values from being sent it the payload by providing an array of key names:

```javascript
Raygun.filterSensitiveData(['password', 'credit_card']);
```

By default this is applied to the UserCustomData object only (legacy behavior). To apply this to any key-value pair, you can change the filtering scope:

```javascript
// Filter any key in the payload
Raygun.setFilterScope('all');

// Just filter the custom data (default)
Raygun.setFilterScope('customData');
```

You can also pass RegExp objects in the array to `filterSensitiveData`, for fuzzy matching of keys:

```javascript
// Remove any keys that begin with 'credit'
var creditCardDataRegex = /credit\D*/;
Raygun.filterSensitiveData([creditCardDataRegex]);
```

### Source maps support

Raygun4JS now features source maps support through the transmission of column numbers for errors, where available. This is confirmed to work in recent version of Chrome, Safari and Opera, and IE 10 and 11. See the Raygun dashboard or documentation for more information.

### Offline saving

The provider has a feature where if errors are caught when there is no network activity they can be saved (in Local Storage). When an error arrives and connectivity is regained, previously saved errors are then sent. This is useful in environments like WinJS, where a mobile device's internet connection is not constant.

### Errors in scripts on other domains

Browsers have varying behavior for errors that occur in scripts located on domains that are not the origin. Many of these will be listed in Raygun as 'Script Error', or will contain junk stack traces. You can filter out these errors by settings this:

```javascript
Raygun.init('apikey', { ignore3rdPartyErrors: true });
```

There is also an option to whitelist domains which you **do** want to allow transmission of errors to Raygun, which accepts the domains as an array of strings:

```javascript
Raygun.init('apikey', { ignore3rdPartyErrors: true }).whitelistCrossOriginDomains(["jquery.com"]);
```

This can be used to allow errors from remote sites and CDNs.

The provider will default to attempt to send errors from subdomains - for instance if the page is loaded from foo.com, and a script is loaded from cdn.foo.com, that error will be transmitted on a best-effort basis.

To get full stack traces from cross-origin domains or subdomains, these requirements should be met:

* The remote domain should have `Access-Control-Allow-Origin` set (to include the domain where raygun4js is loaded from).

* For Chrome the `script` tag must also have `crossOrigin="Anonymous"` set.

* Recent versions of Firefox (>= 31) will transmit errors from remote domains will full stack traces if the header is set (`crossOrigin` on script tag not needed).

In Chrome, if the origin script tag and remote domain do not meet these requirements the cross-origin error will not be sent.

Other browsers may send on a best-effort basis (version dependent) if some data is available but potentially without a useful stacktrace. The provider will cancel the send if no data is available.

#### Options

Offline saving is **disabled by default.** To get or set this option, call the following after your init() call:

```javascript
Raygun.saveIfOffline(boolean)
```

If an error is caught and no network connectivity is available (the Raygun API cannot be reached), or if the request times out after 10s, the error will be saved to LocalStorage. This is confirmed to work on Chrome, Firefox, IE10/11, Opera and WinJS.

Limited support is available for IE 8 and 9 - errors will only be saved if the request times out.

## Release History

[View the changelog here](CHANGELOG.md)
