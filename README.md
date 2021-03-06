# leaflet wms animator [![npm version](https://badge.fury.io/js/leaflet-wms-animator.svg)](https://badge.fury.io/js/leaflet-wms-animator)

Animate WMS layers with temporal dimensions.

Some WMS implementations now support animations through the use of animated GIFs 
(e.g. [Geoserver WMS Animator](http://docs.geoserver.org/stable/en/user/tutorials/animreflector.html)).<br/>
However; the lack of start/stop/step/rewind etc. functionality in a GIF limits the usefulness of this approach.

This simple JS plugin provides some convenience functions to pre-fetch a collection of temporal slices 
from WMS to step through and/or animate them as [leaflet image overlays](http://leafletjs.com/reference.html#imageoverlay).

Note that this plugin works for ncWMS (as per example, `params` object accepts arbitrary key/value pairs).

## notes before use

- **Please use responsibly** if you you attempt to request too many tiles at once, you may cause out of memory issues in your target WMS server.
 A better approach for larger animations is to use the `frames` param to supply images you have pre-cached yourself.
- To get around CORS restrictions, I am using a proxy server. If using this plugin to generate your own frames - you will also need a proxy server, OR have admin access to your target WMS to enable CORS.
- This plugin uses [ES6 Promise](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise) syntax,
 which is [not supported](http://caniuse.com/#feat=promises) by older browsers - you may need a [Polyfill](https://babeljs.io/docs/usage/polyfill/).

## example use
```javascript
var args = {
		
	// reference to your leaflet map
	map: map,
	
	// WMS endpoint
	url: 'http://localhost:8080/geoserver/wms',
	
	// time slices to create (u probably want more than 2)
	times: ["2016-09-17T11:00:00.000Z", "2016-09-17T12:00:00.000Z"],
	
	// the bounds for the entire target WMS layer
	bbox: ["144.9497022","-42.5917177","145.7445272","-41.9883032"],
	
	// how long to show each frame in the animation  
	timeoutMs: 300,
	
	// PREFERRED: provide your own frames, that have been cached earlier:
	frames: [
		{
			"time": "2016-09-17T11:00:00.000Z",
			"img": <base64 string>
		},
		{
            "time": "2016-09-17T12:00:00.000Z",
            "img": <base64 string>
        },
        
        ...
	],
	
	
	// OPTIONAL - only required if you are not providing your own frames
	// **See defining image request for more info**
	// due to CORS restrictions, you need to define an async function to ask your proxy server to make the WMS 
	// GetMap request and resolve the result (as a base64 encoded string). This example is using a call to a server function called 
	// 'getImage' (in MeteorJS). Note that if your target WMS is CORS enabled, you can just define a direct HTTP request here instead.
	proxyFunction: function(requestUrl, time, resolve, reject){
		
		Meteor.call('getImage', requestUrl, function(err, base64ImgString) {
			if(err){
				reject(err);
			}

			resolve({ time: time, img: base64ImgString });
		});
	},
	
	// OPTIONAL - only required if you are not providing your own frames
	// your WMS query params
	params: {
		BBOX: "144.9497022,-42.5917177,145.7445272,-41.9883032",
		LAYERS: "temp",
		SRS: "EPSG:4326",
		VERSION: "1.1.1",
		WIDTH: 2048, 
		HEIGHT: 2048,
		transparent: true,

		// ncWMS params (optional)
		abovemaxcolor: "extend",
		belowmincolor: "extend",
		colorscalerange: "10.839295,13.386014",
		elevation: "-5.050000000000001",
		format: "image/png",
		logscale: false,
		numcolorbands: "50",
		opacity: "100",
		styles: "boxfill/rainbow"
	}
};

LeafletWmsAnimator.initAnimation(args, function(frames){

	// if you didn't provide your own frames this callback function returns the 
	// array of images with their respective time stamps (e.g. you can use timestamps in UI)
});
```

## defining the image request

Images for the map layers are defined as [base64](https://en.wikipedia.org/wiki/Base64) strings. 

For simplicity, in my proxy server function - I use the `encode` method from 
[node-base64-image](https://www.npmjs.com/package/node-base64-image) wrapped in a Promise, like this:

```javascript
return new Promise((resolve, reject) => {
	encode(url, {string: true}, function (err, res) {
		if(err) reject(err);
		
		// returns a base64 encoded string representing our layer image
		resolve('data:image/png;base64,' + res);
	});
});
```

## convenience functions

- <strong>forward:</strong> step forward to next frame
- <strong>backward:</strong> step backward to previous frame
- <strong>play:</strong> start animating
- <strong>pause:</strong> pause animation
- <strong>setFrameIndex:</strong> skip to a specific animation frame
- <strong>destroyAnimation:</strong> destroy animation, removes image overlay layers from map etc.

## events

The `wmsAnimatorFrameIndexEvent` is dispatched from `window` every time a frame is changed, you can listen to this event to know what time frame is currently active.

Example use:

```javascript
window.addEventListener('wmsAnimatorFrameIndexEvent', function (e) {
   console.log('current frame time is at array index: '+ e.detail);
});
```

## license
MIT
