# iNaturalist Live
Live visualisation of iNaturalist observations and identifications.

![image](https://user-images.githubusercontent.com/17750766/138076782-04740f0e-a670-43dc-af5c-07efafdf2411.png)

## About

This is a one-page serverless web page that calls the iNaturalist API: https://api.inaturalist.org/v1/docs/ to show actions across the whole of iNaturalist such as submission of records and identification of records.

It has three main features:
 * A central counter for the number of verifable records in iNaturalist
 * A map with markers for new submissions or identification on iNatauralist. Markers have a popup with a small image of the observation, and the current identification. THe symbol of the marker indicates whether it is an observation or a identification, and the type of identification, as idicated by the key on the bottom of the page.
 * A 'ticker' on the right hand side which lists new observations / identifications as they are added to iNaturalist. Mouse rollover highlights the corresponding marker on the map. Clicking on the ticker item takes you to inaturalist to look at the observation on iNaturalist.org.

Visit the webpage: https://simonrolph.github.io/inatcounter/

Note that each page you have open is calling the iNaturalist API individually so 

## How it works

### Set up

All the html and JS is in a single file: https://github.com/simonrolph/inatcounter/blob/master/index.html and css is in: https://github.com/simonrolph/inatcounter/blob/master/style.css

The map is a leaflet map: https://leafletjs.com/

The counter ticks up (or down) using countUp.js https://inorganik.github.io/countUp.js/

Custom leaflet map markers are defined using font-awesome (https://fontawesome.com/) icons. For example this is the marker for observations:
```javascript
var obs = new L.DivIcon({
    className: 'my-div-icon',
    html: '<span class="fa-stack"><i class="fas fa-circle fa-stack-2x fa-inverse" style="text-shadow: 0 0 3px rgba(0, 0, 0, 0.2);"></i><i class="fas fa-camera fa-stack-1x"></i></span>'
});
```

Firstly a function gets the current date/time for x number of seconds ago (as specified in the `secondsdiff` argument) and formats it in the right string format for calling the iNaturalist API.

```javascript
function getTheDate(secondsdiff) {
    var d = Math.floor((new Date()).getTime() / 1000) + secondsdiff
    //console.log(d);
    var da = new Date(0); // The 0 there is the key, which sets the date to the epoch
    da.setUTCSeconds(d);

    // make the date in the right format for the API call

    // example correct format: 2018-10-14T12%3A44%3A24%2B00%3A00
    var outDate = String(addZero(da.getUTCFullYear())) + "-" + String(addZero(da.getUTCMonth() + 1)) + "-" + String(addZero(da.getUTCDate())) + "T" + String(addZero(da.getUTCHours())) + "%3A" + String(addZero(da.getUTCMinutes())) + "%3A" + String(addZero(da.getUTCSeconds())) + "%2B00%3A00";

    return (outDate)
}
```

### API calls

The page uses the `window.setInterval(function(){}, 15000)` to make API calls at set intervals in milliseconds, so `15000` = 15 seconds. The page then makes 3 different calls to the API at various intervals.

Every 15 seconds it updates the big counter in the middle 

```javascript
var theDate = getTheDate(0)
request.open('GET', 'https://api.inaturalist.org/v1/observations?verifiable=true&created_d2=' + theDate + '&page=1&per_page=0&order=desc&order_by=created_at', true);
```

Every 6 seconds it gets recent observations from the past 10 seconds using an API call (limited to one page of 40 so if someone uploads a massive batch then it might miss some of them) like so:

```javascript
var theDate2 = getTheDate(-10);
request2.open('GET', 'https://api.inaturalist.org/v1/observations?verifiable=true&created_d1=' + theDate2 + '&page=1&per_page=40&order=desc&order_by=created_at', true);
```

It then uses this data to update the map and ticker. In the same interval it gets identifications from the past 10 seconds like so:

```javascript
request3.open('GET', 'https://api.inaturalist.org/v1/identifications?current=true&d1=' + theDate2 + '&order=desc&order_by=created_at', true);
```

Since the call is getting observations/identifications from the last 10 seconds but doing the action every 6 seconds (to accomodate time for the api call to be made) the API might get pull the same observation/identification twice. Therefor, it builds up an array of unique IDs of observations / identifications in the oject `uuid` and for each new item in pulled data it checks if it's already downloaded that item before.

5 minutes after it has added the markers/ticker items, it then removes the markers / ticker items using this code:
```
window.setTimeout(function(markerid) {
  //console.log(markerid);
  document.getElementById("marker"+markerid).remove();
  mymap.removeLayer(layerObvs.getLayer(markerid));

},300000,marker.marker_id);
```

That's essentially it really. There are various comments in the code which point out what each bit is doing: https://github.com/simonrolph/inatcounter/blob/master/index.html

If I were to make it again I'd make it have some server-side operation that pulls the data from the API once per time interval, and then the webpage is served by that data - rather than each instance of someone having the web page open contacting the iNaturalist API directly - because then it wouldn't be at risk of overloading their API servers.
