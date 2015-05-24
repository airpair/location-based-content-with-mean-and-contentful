At [We Are Living](http://weareliving.co.uk/#/), we spent quite a bit of time playing with iBeacons, HTML geolocation and new mapping technologies. After discovering a generous selection of tools for working with location based data, we decided to build [Leeds Living](http://leedsliving.co.uk/), a digital city guide with a focus on location and real-time data. So to demonstrate how we achieved some of the core functionality within Leeds Living, we will share a taster of the process behind the product. This article assumes you already have some knowledge of all technologies mentioned.

## Choosing a CMS
By the end of 2015, we’re hoping to unveil a new app - essentially Leeds Living on steroids. [The Living App](http://thelivingapp.com/#/) will be a groundbreaking lifestyle app integrated with thousands of iBeacons installed across Leeds city centre, providing real-time anonymous marketing data to businesses and pushing offers and useful information to users when they are out and about in the city. So we needed a CMS that would not only deliver content to the Leeds Living site, but also future-proof us for when we need to access the same data from the app. We chose [Contentful](https://www.contentful.com/) for this very reason. The guys that built it in Berlin call it an “API-first content management system for multi-device online publishing”.

When building content types within the system, we came across 2 fields that were simple yet invaluable to us. The ‘location’ field accepts a postcode or address and stores it as coordinates. The ‘object’ field accepts an arbitrary JSON structure; we used this to house more complex location information.

## Content Delivery
We began entering local businesses into the system, each with location coordinates. The [content delivery API](https://www.contentful.com/developers/documentation/content-delivery-api/) then allowed us to access this content from our NodeJS server.

```javascript,linenums=true
var contentful = require("contentful").createClient({
  accessToken: "xxxxxxxx",
  space: "xxxxxxxx"
});

app.get("/api/place/:slug", function(req, res){ 
 contentful.entries({ 
    "content_type": contentType.place,
    "fields.slug[in]": req.params.slug,
    "limit": 1
  }, function(err, entries){
    res.send(entry[0]);
  }); 
});
```

This API request is made by our Angular frontend, where we also apply the results to our scope.

```javascript,linenums=true
app.controller("PlaceController", function($scope, $routeParams, $http){ 
  $http.get("/api/place/" + $routeParams.slug).then(function(res){
    $scope.place =  res.data;
  });
});
```

The content can now be rendered in a browser as HTML. Easy as pie!

```javascript,linenums=true
<div>{{place.fields.name}}</div>
<div>{{place.fields.about}}</div>
<a ng-href="mailto:{{place.fields.email}}">Make enquiry</a>
<a ng-href="{{place.fields.website}}" target="_blank">Website</a>
```

## Dynamic Mapping

The beauty here is that we also have the coordinates for each place stored in Contentful. So we used the formidable mapping solution [Mapbox](https://www.mapbox.com/) to display a map on the page with a marker showing the location of the place. We simply add to our controller:

```javascript,linenums=true
$http.get("/api/place/" + $routeParams.slug).then(function(res){
  $scope.place =  res.data;
  buildMap();
});

var buildMap = function(){

  L.mapbox.accessToken = "xxxxxxxx";
  var map = L.mapbox.map("map", "xxxxxxxx" {
    minZoom: 11,
    zoomControl: false,
    attributionControl: false
  }).setView([53.80451586014786, -1.5477633476257324], 15, {
    pan: { animate: true },
    zoom: { animate: true } 
  });
    
  L.mapbox.featureLayer({
    type: "Feature",
    geometry: {
      type: "Point",
      coordinates: [$scope.place.fields.location.lon, $scope.place.fields.location.lat]
    },
    properties: { "title": $scope.place.fields.name }
  }).addTo(map);

}
```

We took this one step further and added a button to our map, which would allow the user to see their current location in relation to the place (if they’re in the city area). This is where [HTML geolocation](http://diveintohtml5.info/geolocation.html) comes in handy. Firstly, we ask the user to share their location when they visit the site.

```javascript,linenums=true
if (navigator.geolocation) {
  navigator.geolocation.getCurrentPosition(function(position){      
    $rootScope.geolocation = position;
  });
}
```

Then we add our HTML button to the map.

```javascript,linenums=true
<a href="" ng-click="showMe()"></a>
```
And simply add another feature layer to the map when the button is clicked, showing a marker where the user is.

```javascript,linenums=true
$scope.showMe = function(){
  L.mapbox.featureLayer({
    type: "Feature",
    geometry: {
      type: "Point",
      coordinates: [$rootScope.geolocation.coords.longitude, $rootScope.geolocation.coords.latitude]
    },
    properties: { "title": "You are here" }
  });
});
```

Of course there’s a bit more to it than that. We actually handled all map related processing in an [Angular service](https://docs.angularjs.org/guide/services) and added some additional features such as zooming the map to [fit both markers within the canvas](https://www.mapbox.com/mapbox.js/example/v1.0.0/fit-map-to-markers/) when the ‘show me’ button is clicked.

## Location Based Suggestions

During our R&D phase, we stumbled across a variety of city guide websites, all doing similar things. If you visit one of these sites looking for the best local burger place, you’ll most likely also find a ‘suggested places’ box filled with alternative burger places or relatively useless ‘author’s picks’. We thought we could do better. What if we suggested other nearby places? The user would look up the burger place and also get suggestions for shops and bars within a short walk. Now we’re talking! You can now easily plan your whole afternoon or evening in the city.

This is where [search parameters](https://www.contentful.com/developers/documentation/content-delivery-api/javascript/#search) of the Contentful API had their time to shine. The clever boys and girls at Contentful built location based search into their content delivery API, with options to find ‘entries near me’, ‘search within bounding rectangle’ or ‘search within bounding circle’.

So to find nearby places, we simply add to our Angular controller:

```javascript,linenums=true
$http.get("/api/place/" + $routeParams.slug).then(function(res){
  $scope.place =  res.data;
  buildMap();
  nearbyPlaces();
});

var nearbyPlaces = function(){
  $http.post("/api/place/nearby", {
    lat: $scope.place.location.lat, 
    lon: $scope.place.location.lon
  }).then(function(res){      
    $scope.nearbyPlaces = res.data;
  });
}
```
Then to request the search through our NodeJS server:

```javascript,linenums=true
app("/api/place/nearby", function(req, res){
  contentful.entries({
    "content_type": "xxxxxxxx",
    "fields.location[near]": req.body.lat + "," + req.body.lon,
    "limit": 8
  }, function(err, entries){
    res.send(entries);
  });
});
```
Once again the results can be easily rendered in a browser using Angular. For groups of data like this, we often opted to use [ngRepeat](https://docs.angularjs.org/api/ng/directive/ngRepeat).

```javascript,linenums=true
<a ng-repeat="nearbyPlace in nearbyPlaces" ng-href="/places/{{nearbyPlace.fields.slug}}">
  <span>{{nearbyPlace.fields.name}}</span>
  <span>{{nearbyPlace.fields.description}}</span>
</a>
```

## That’s All Folks!

So there you have it! We still have a long road ahead for improving the functionality on Leeds Living and developing it further. We want to introduce more location features such as a ‘show places near me’ button and site filtering such as ‘show restaurants open near me’. And it’s not just places that use the location data. Some of our articles also use the map and show related places based on relevance or location.

Ultimately we’ve opened the flood gates here and the possibilities are endless. We think this is the future for city guides and with the help of well supported tools such as Contentful and Mapbox, we can make this possible. 