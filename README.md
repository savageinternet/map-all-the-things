# Map all the Things with Mapbox GL!

This repo contains source code and step-by-step instructions for a workshop entitled _Map all the Things with Mapbox GL!_  This was first presented as a <a href="https://candu.github.io/presentations/html/20200527-fwdjs-map-all-the-things">talk</a>at <a href="https://forwardjs.com/ottawa/schedule/">ForwardJS Ottawa 2020</a> to a virtual audience of roughly 700.  Given the high level of interest, I decided to submit it as a longer workshop for <a href="https://www.startupslam.io/2020">Startup Slam 2020</a>, and here we are!

## Session Description

Come learn how to map millions of features while keeping your visualizations understandable and navigable for your users!  In this session, we'll use Mapbox GL and open data from the City of Toronto to cover tools and techniques for mapping large datasets.  From Mapbox GL basics to vector tiles, heatmaps, and clustering, you'll learn several ways to harness the power of these datasets in your applications.

Although we'll focus on web-based applications, many of the same techniques can be used alongside Mapbox mobile SDKs to power map-based experiences on mobile.

## Session Requirements

You'll get more out of this session if you have some experience with frontend (HTML / CSS / JavaScript) development and are comfortable using npm or similar JavaScript package managers.  Previous experience with mapping and/or data visualization also helps, but is not required.  Slides and materials will be available after the workshop as well, so you'll be able to refer back to them anytime.

# Getting Started

```bash
git clone https://github.com/savageinternet/map-all-the-things.git
cd map-all-the-things

git checkout start-here

npm install
npm run serve
```

This repository also includes a `.code-workspace` file for use with Visual Studio Code, as well as an `.nvmrc` file in case you're using `nvm`.

# Step-by-Step

## Step 0: Starting page

We have a simple full-page layout, plus a bit of scaffolding for Mapbox
GL that we aren't yet using.

To get to the next step, you have to clean up some of the
eye-searing-ness and initialize the map properly!

Let's start by getting rid of that `<h1>` element:

```html
<div id="map_container"></div>
```

Like other mapping libraries, Mapbox GL needs a _container element_ for its map.  In our case, that's `#map_container`, which has been styled in CSS to take up the whole page width and height.  We get a reference to this container, then attach it in options and initialize the map:

```js
function initMap() {
  const $mapContainer = document.getElementById('map_container');
  // or: document.querySelector('#map_container')

  const options = {
    // set the container in options
    container: $mapContainer,
    dragRotate: false,
    pitchWithRotate: false,
    renderWorldCopies: true,
    style: basemapStyle,
  };

  // finally, create a new `mapboxgl.Map` instance using these options!
  map = new mapboxgl.Map(options);
}
```

Congratulations!  You've created your first Mapbox GL map.  We'll be
building on this map incrementally over the rest of the workshop.

## Step 1: Toronto map

Now that we've shown our first map, let's focus on a specific area.
Since we'll be using City of Toronto data, it makes sense to focus on
Toronto.

To get to the next step, you have to configure Mapbox GL to stay within
Toronto city bounds!

First, we need a _bounding box_ for Toronto:

```js
const BOUNDS_TORONTO = new mapboxgl.LngLatBounds(
  new mapboxgl.LngLat(-79.639264937, 43.580995995),
  new mapboxgl.LngLat(-79.115243191, 43.855457183),
);
```

But wait!  The map defaults to _zoom level 0_, which shows the entire world.  We don't need to zoom out that far.  By default, it also allows the user to zoom in as far as _zoom level 23_ - that's _way_ too close for our purposes.  In addition to our bounding box, then, we need some _zoom bounds_:

```js
const ZOOM_LEVEL_0 = 10;
const ZOOM_LEVEL_3 = 19;
```

Why `ZOOM_LEVEL_0` and `ZOOM_LEVEL_3`?  Similar to breakpoints in responsive design, these can be thought of as _zoom breakpoints_.  We'll be adding more breakpoints between these two.

We can then use the bounding box and zoom bounds to keep the viewer within Toronto city limits:

```js
function initMap() {
  // ...
  const options = {
    // add in these options:
    center: BOUNDS_TORONTO.getCenter(),
    maxBounds: BOUNDS_TORONTO,
    minZoom: ZOOM_LEVEL_0,
    maxZoom: ZOOM_LEVEL_3,
    zoom: ZOOM_LEVEL_0,
    // change this line (optional, but we don't need it to be `true`):
    renderWorldCopies: false,
    // ...
  };
  // ...
}
```

Note that we've also set `center` and `zoom` - these tell Mapbox GL where the view should initially be placed.  (This is similar to the concept of a _camera_ in graphics / game programming: it's one particular view of a larger world!)

Great!  By using bounding box and zoom limiting options, you've focused
the map on Toronto.  This puts us in a good place to start adding City
of Toronto data.

## Step 2: GeoJSON data layers

### Step 2a: loading collision data

With the map focused on Toronto, we're ready to add some City of Toronto
data!  To do that, though, we first have to load the data.

To get to the next step, you have to load collision data into our page,
then test that the data is loading properly.

You might have noticed that the page we're building comes equipped with a function called `getJson()`:

```js
async function getJson(url) {
  const response = await fetch(url);
  return response.json();
}
```

This function takes a URL and returns a `Promise` that resolves to the JSON response from issuing an HTTP `GET` request to that URL.

You might _also_ have noticed another file in the `src/data` folder in this repository:

```bash
$ ls data
collisions.geojson metadata.json      root.json
```

We're already using `metadata.json` and `root.json` to create the _basemap_ - these are borrowed from the [World Dark Gray Base](https://www.arcgis.com/home/item.html?id=a284a9b99b3446a3910d4144a50990f6) tile layer made available by [ESRI](https://esri.maps.arcgis.com/home/index.html).

That's right!  Despite what the [Mapbox GL docs](https://docs.mapbox.com/mapbox-gl-js/api/) say, you _do not need_ a Mapbox access token to use Mapbox GL, nor do you need to use a map style hosted with Mapbox.  You can use any style from anywhere, so long as it conforms to the [Mapbox GL Style Specification](https://docs.mapbox.com/mapbox-gl-js/style-spec/).

But enough about basemaps - we want data to put on the basemap!  That's where `collisions.geojson` comes in.  Let's add that to the datasets we fetch in `initMapbox()`:

```js
let collisions = null;

async function initMapbox() {
  const [
    collisionsData,  // <--
    metadata,
    root,
  ] = await Promise.all([
    getJson('data/collisions.geojson'), // <--
    getJson('data/metadata.json'),
    getJson('data/root.json'),
  ]);

  basemapStyle = getStyle(metadata, root);
  collisions = collisionsData; // <--
}
```

Before we start using this data, let's check that it actually loads successfully.  To do this, we'll add an _event listener_ for the `load` event on the `mapboxgl.Map` object:

```js
function initMap() {
  // ...

  map = new mapboxgl.Map(options);
  map.on('load', populateMap);
}

function populateMap() {
  console.log(collisions);
}
```

From the [Mapbox GL docs on `load`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map.event:load):

> Fired immediately after all necessary resources have been downloaded and the first visually complete rendering of the map has occurred.

Awesome!  You've loaded collision data from the City of Toronto, and can
see that we have roughly 175 000 points.  (The full dataset we have is
much larger, but this is large enough to show off some of the techniques
here.)  Now we can use that data in the map.

### Step 2b: using collision data

OK, we've got collision data loaded into the page, ready to use in our
map.  In Mapbox GL, data is provided by _sources_, which are then used
within _layers_ to render that data to the map.

To get to the next step, you have to add a GeoJSON source using the
newly loaded collisions data, then add a layer that uses that source.

We add sources using `map.addSource()`:

```js
function populateMap() {
  // don't need the `console.log()` anymore
  map.addSource('collisions', {
    type: 'geojson',
    data: collisions,
    buffer: 0,
  });
}
```

That's it!  Under the covers, Mapbox GL takes our 30 MB collisions dataset and indexes it into vector tiles, _right in the browser_.  (We don't _need_ the `buffer` option, but for `circle` layers it can speed up both vector tile indexing and subsequent rendering.)

From there, we use sources in layers by using the `source` option:

```js
function populateMap() {
  // ...
  map.addLayer({
    id: 'collisionsPoints',
    source: 'collisions',
    type: 'circle',
    minzoom: ZOOM_LEVEL_0,
    maxzoom: ZOOM_LEVEL_3,
    paint: {
      'circle-color': '#ef4848',
      'circle-radius': 6.5,
      'circle-stroke-color': '#773333',
      'circle-stroke-width': 1,
    },
  });
}
```

The `paint` options tell Mapbox GL how to style these circles.

Whoa, that's a lot of data.  You've used the collision data in your
first source and layer to show each collision as its own point.  It's a
good start, but we can do a *lot* more to make it easier for users to
understand and navigate this dataset!

## Step 3: Heatmaps

### Step 3a: switching to heatmap

So we have a lot of data on a map, but we can't make sense of it!
Fortunately, Mapbox GL has several features that help us visualize large
datasets.  The first one we'll look at is _heatmap layers_: it's easy to
configure a layer to show its data not as a series of individual points,
but as a heatmap.

To get to the next step, you have to update your layer configuration to
turn it into a heatmap.

First, though, let's pull our fill and stroke colors from before up into variables, so that we can reuse them throughout this workshop:

```js
const COLOR_COLLISION_FILL = '#ef4848';
const COLOR_COLLISION_STROKE = '#773333';
```

Now we can adjust our layer as follows:

```js
map.addLayer({
  id: 'collisionsHeatmap',
  source: 'collisions',
  type: 'heatmap',
  minzoom: ZOOM_LEVEL_0,
  maxzoom: ZOOM_LEVEL_3,
  paint: {
    'heatmap-color': COLOR_COLLISION_FILL,
    'heatmap-weight': 0.03,
  },
});
```

Oh no!  You've traded out your beautiful circles for a uniform sea of
eye-searing red.  Never fear, though: we'll work this into a usable
heatmap over the next couple of substeps.

### Step 3b: heatmap color ramps

We can fix this sea of red by using Mapbox GL _style expressions_, which
let us apply data-driven styling to map layers.  We'll tweak other
things in a moment, but for now we'll focus our attention on
`heatmap-color`.

To get to the next step, you have to use style expressions to create a
nice color ramp for your heatmap.

Let's start with the color ramp.  Replace the value of the `heatmap-color` property with this expression:

```js
[
  'interpolate',
  ['linear'],
  ['heatmap-density'],
  0, COLOR_COLLISION_HEATMAP_ZERO,
  0.5, COLOR_COLLISION_HEATMAP_HALF,
  1, COLOR_COLLISION_FILL,
]
```

To get this working, we'll also need to define two new colors:

```js
const COLOR_COLLISION_HEATMAP_ZERO = 'rgba(244, 227, 219, 0)';
const COLOR_COLLISION_HEATMAP_HALF = '#f39268';
```

You can use these values or play around with your own.  The only requirement is that `COLOR_COLLISION_HEATMAP_ZERO` have an _alpha_ value of `0` - this is what allows the heatmap to fade to transparent wherever there aren't any points.

OK, let's pick this apart a bit.  This is your first real look at Mapbox GL _style expressions_!  If you've ever worked with [Scheme / Racket](https://racket-lang.org/), [Clojure](https://clojure.org/), or other LISP-like languages, these expressions will look familiar: they're basically S-expressions in JSON array form.

For the rest of us, though, that's not a helpful description!  A _style expression_ can be a value, as in `'#ef4848'` or `6.5`:

```js
{
  'circle-color': '#ef4848',
  'circle-radius': 6.5,
  'circle-stroke-color': '#773333',
  'circle-stroke-width': 1,
}
```

It can also be an array, as in our heatmap example:

```js
[
  'interpolate',
  ['linear'],
  ['heatmap-density'],
  0, COLOR_COLLISION_HEATMAP_ZERO,
  0.5, COLOR_COLLISION_HEATMAP_HALF,
  1, COLOR_COLLISION_FILL,
]
```

The first element of this array is an _operator_, and everything else is an _argument_ to that operator.  So `interpolate` is the operator, and from [the `interpolate` documentation](https://docs.mapbox.com/mapbox-gl-js/style-spec/expressions/#interpolate) we can see what it does:

> Produces continuous, smooth results by interpolating between pairs of input and output values ("stops").

The first argument is the `interpolation` function: here we're using `['linear']`, which is just a simple linear ramp.  (You can also use exponential or Bézier curves.)

The next one, `['heatmap-density']`, is the `input`.  This is another array expression!  `heatmap-density` is a special operator that returns the _density_ of the heatmap at each pixel.  (Explaining _density_ here is beyond the scope of this workshop, but in layperson terms it's as though you blurred each point, and then added those blurred points up.)

Next is a series of pairs of input and output values - these are the _stops_ mentioned in the documentation.  In our example, a density of `0.25` would map to a color halfway between `COLOR_COLLISION_HEATMAP_ZERO` and `COLOR_COLLISION_HEATMAP_HALF`.  (You could add even more stops and make rainbow scales, or tritone scales, or any of the [ColorBrewer](https://colorbrewer2.org/#type=sequential&scheme=BuGn&n=3) scales...)

And that's it!  `interpolate` uses an `interpolator` to map an `input` value onto an `output` range, using the _stops_ configured in the expression.  In our case, we're using it to map heatmap density (the `input`) onto a color (the `output`).  We'll see several more examples of style expressions as we go along.

Looking better now!  You can actually make out areas of higher and lower
collision density now.  It's a bit blobby-looking, though, so we'll keep
tweaking the style.

### Step 3c: intensity and radius

To help de-blobify our heatmap, we can adjust `heatmap-intensity` and
`heatmap-radius` to values that better suit our data.  There isn't a
hard and fast rule here - you'll have to experiment with your dataset
(and users!) to see what values are most useful.  To make things
interesting, we'll use `interpolate` again, this time with `zoom` to
compensate for points spreading out as we zoom in.

To get to the next step, you have to use more style expressions to set
`heatmap-intensity` and `heatmap-radius`.

To start with, we'll define another _zoom breakpoint_ at zoom level 14:

```js
const ZOOM_LEVEL_1 = 14;
```

Now we can use this new breakpoint in a style expression to vary the `heatmap-intensity` and `heatmap-radius`:

```js
'heatmap-intensity': [
  'interpolate',
  ['linear'],
  ['zoom'],
  ZOOM_LEVEL_0, 0.33,
  ZOOM_LEVEL_1, 1,
],
'heatmap-radius': [
  'interpolate',
  ['linear'],
  ['zoom'],
  ZOOM_LEVEL_0, 5,
  ZOOM_LEVEL_1, 10,
],
```

There's `interpolate` again, this time with `['zoom']` as the `input`.  The `zoom` operator here returns the current map zoom level, so we can use it to change the style gradually as the user zooms in and out!  This is super-powerful: as we'll see again, this allows you to present different views of the same dataset at different scales.

Great!  By reducing the visual noise of your heatmap, you've helped your
users make sense of the data.  It's now much easier to see which areas
have especially high numbers of collisions.

### Step 3d: point weighting in heatmaps

Some collisions are more serious than others.  In the
`collisions.geojson` dataset, each collision has a property called
`ksi`.  This stands for "killed or seriously injured": these are
especially serious collisions that result in hospitalization or even
death.

As a Vision Zero city, Toronto's goal is to treat every single KSI
collision as preventable.  That means improving everything from signage
to lane markings to signal timings / phases to visibility, all in an
attempt to create a safe environment for all road users.  Since
pedestrians, cyclists, seniors, and other vulnerable road users are
overrepresented in KSI statistics, their safety is given the highest
priority.

To get to the next step, you have to give KSI collisions the importance
they deserve by _dramatically_ increasing their weight.

Set `heatmap-weight` as follows:

```js
'heatmap-weight': [
  'case',
  ['get', 'ksi'], 3,
  0.03,
],
```

OK!  We've got two new operators in here: `case` and `get`.  `case` is Mapbox GL's if-then-else block; we can read this expression as "if `['get', 'ksi']`, then return `3`, else return `0.03`".  `get` is a _property accessor_ operator: for each data point, `['get', 'ksi']` returns the value of the `ksi` property.  To see what that means, we can take a quick look at `collisions.geojson`:

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "id": 1742504,
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [
          -79.448825,
          43.638223
        ]
      },
      "properties": {
        "ksi": false,
        "cyclist": false,
        "pedestrian": false
      }
    },
    // ...
```

`['get', 'ksi']` pulls out that `ksi` property.  We could just as easily do `['get', 'cyclist']`, or `['get', 'pedestrian']`, depending on what we want to show in our map.

Phew - you made it.  By tweaking the various parameters of heatmap
layers, you've created a heatmap that better represents the data, and
that appropriately draws attention to the most serious collisions.
We're not done, though!  This heatmap is cool, but we can _still_
improve upon this.

## Step 4: Clustering and Visual Differentiation

### Step 4a: heatmap and points!

Heatmaps are great for seeing the big picture, but not so useful for
getting a precise view of individual data points...so why not use both?

To get to the next step, you have to set up another layer for points,
then use style expressions to crossfade between the heatmap and
individual points as the user zooms in.

When a DJ crossfades two tracks, there's a period of time where both tracks are playing.  Similarly, to crossfade over zoom levels, we need a zoom level where both the heatmap and the points are visible!  To do this, we'll show the heatmap for one additional zoom level by changing `maxzoom`:

```js
maxzoom: ZOOM_LEVEL_1 + 1,
```

Now we need the _fade_ part, for which we'll use `heatmap-opacity`:

```js
'heatmap-opacity': [
  'interpolate',
  ['linear'],
  ['zoom'],
  ZOOM_LEVEL_1, 1,
  ZOOM_LEVEL_1 + 1, 0,
],
```

There's that `interpolate` / `zoom` combo again!  OK, so we're fading out our heatmap, but what are we fading in?  We need a second layer for our points:

```js
map.addLayer({
  id: 'collisionsPoints',
  source: 'collisions',
  type: 'circle',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  paint: {
    'circle-color': COLOR_COLLISION_FILL,
    'circle-opacity': [
      'interpolate',
      ['linear'],
      ['zoom'],
      ZOOM_LEVEL_1, 0,
      ZOOM_LEVEL_1 + 1, 1,
    ],
    'circle-radius': 6.5,
    'circle-stroke-color': COLOR_COLLISION_STROKE,
    'circle-stroke-width': 1,
  },
});
```

And there's the fade in effect!  Between `ZOOM_LEVEL_1` and `ZOOM_LEVEL_1 + 1`, our heatmap will fade out while our points fade in.

Yay!  You made your first map with more than one layer, and you've even
set up a cool crossfade effect between those layers as you zoom in.
This is one of the great things about interactive maps: you can show
more than one representation of the data to help users see both the big
picture _and_ the details.

### Step 4b: clustering

Even with our new crossfade technique in place, we're still showing a
_lot_ of visual clutter with all those points.  To handle large
point-based datasets, Mapbox GL also supports _clustering_.  We'll use
this feature to help tame visual clutter at higher zoom levels.  It'll
take a bit of work, but it's worth it!

To get to the next step, you have to add a clustered data source for
collisions, then update our points layer to use that data source.

Clustered data sources are another piece of magic available out-of-the-box in Mapbox GL:

```js
map.addSource('collisionsClustered', {
  type: 'geojson',
  data: collisions,
  cluster: true,
  clusterMaxZoom: ZOOM_LEVEL_3,
  clusterRadius: 30,
});
```

Under the covers, this uses a technique called [hierarchical greedy clustering](https://blog.mapbox.com/clustering-millions-of-points-on-a-map-with-supercluster-272046ec5c97), which works quite well with many real-world datasets (like ours!)  We've chosen a `clusterRadius` of `30`, but you should feel free to play with that value!  Lower values mean more clusters, which means more precise locations at the cost of more clutter and lower performance.

Now we're going to update our `collisionsPoints` layer, renaming it `collisionsClustered` in the process.  (You can name these however you like, but as with all programming tasks it's helpful to pick descriptive names!)

```js
map.addLayer({
  id: 'collisionsClustered',
  source: 'collisionsClustered',
  type: 'circle',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  filter: ['has', 'point_count'],
  paint: {
    'circle-color': COLOR_COLLISION_FILL,
    'circle-opacity': [
      'interpolate',
      ['linear'],
      ['zoom'],
      ZOOM_LEVEL_1, 0,
      ZOOM_LEVEL_1 + 1, 1,
    ],
    'circle-radius': [
      'step',
      ['get', 'point_count'],
      8,
      10, 10,
      100, 14,
      1000, 16,
    ],
    'circle-stroke-color': COLOR_COLLISION_STROKE,
    'circle-stroke-width': 1,
  },
});
```

Note the `filter` on this layer: we can use a Mapbox GL style expression here to only show specific points.  When building a clustered data source, Mapbox GL adds the `point_count` property to each cluster - as you might expect, this is the number of points in that cluster!  So this layer will only show _clusters_: individual points that didn't get clustered aren't included.  (We'll get to those in a bit.)

There's also a new operator: `step`.  This is similar to `interpolate`, in that it takes a series of stops that describe how to map `input` values to `output` values.  In this case, though, we're not interpolating between `output` values.  As an example: using our expression above, a cluster with a `point_count` of `55` would have a radius of `10`, whereas if we'd used `interplate` with `['linear']` as the interpolator we'd instead get a radius of `12`.

OK!  You've taken the first step towards moving from individual points
to clusters at higher zoom levels.  We're not done yet, though: we still
need to handle points that _don't_ make it into clusters, and it would
be nice to show the point count on each cluster!

### Step 4c: cluster labels and unclustered points

Now that we have the basic clusters in place, let's finish this off!  In
Mapbox GL, text labels use the `symbol` layer type.  (Icons _also_ use
`symbol`, but we won't cover those in this workshop.)  Since each layer
can only have one type, this means we need a separate layer to show
cluster point counts.

To get to the next step, you have to add two new layers - one for the
cluster point counts, one to handle unclustered single points.

Let's start with the cluster point counts:

```js
map.addLayer({
  id: 'collisionsClusteredCount',
  source: 'collisionsClustered',
  type: 'symbol',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  filter: ['has', 'point_count'],
  layout: {
    'text-field': '{point_count_abbreviated}',
    'text-font': ['literal', ['Ubuntu Regular']],
    'text-size': 12
  },
  paint: {
    'text-color': COLOR_COLLISION_STROKE,
  }
});
```

This is our first layer of type `symbol`.  In Mapbox GL, `symbol` layers are used to show text and/or icons.  (You can actually show both: this is great for showing markers with numbers on them, for instance!)

To show text, we need to provide a `text-field`.  In this case, we're using the `point_count_abbreviated` cluster property - this is similar to `point_count`, except that it's a `string` instead of a `number`, and it abbreviates larger values (e.g. `'2.5k'` instead of `'2513`.)

We also need a `text-font`.  Similar to CSS `font-family`, this takes a _font stack_; each font is tried in sequence until a supported font is found.  But how do we provide an array of fonts?  After all, arrays are _already_ used as expressions.  That's where `literal` comes in: this is an operator that returns its first argument.  As an example:

```js
// this is an error: there's no `Ubuntu Regular` operator!
['Ubuntu Regular']

// this works: `literal` returns its first argument, in this case `['Ubuntu Regular']`
['literal', ['Ubuntu Regular']]
```

There's also `text-size` and `text-color`.  Note that `text-field`, `text-font`, and `text-size` are _layout_ options while `text-color` is a _paint_ option.  If you try to put `text-color` in `layout`, or `text-size` in `paint`, Mapbox GL will give you an error.

Now that we have text labels to show the number of points in each cluster, there's just one last piece to clean up: unclustered points.  They get their own layer:

```js
map.addLayer({
  id: 'collisionsUnclustered',
  source: 'collisionsClustered',
  type: 'circle',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  filter: ['!', ['has', 'point_count']],
  paint: {
    'circle-color': COLOR_COLLISION_FILL,
    'circle-opacity': [
      'interpolate',
      ['linear'],
      ['zoom'],
      ZOOM_LEVEL_1, 0,
      ZOOM_LEVEL_1 + 1, 1,
    ],
    'circle-radius': 4,
    'circle-stroke-color': COLOR_COLLISION_STROKE,
    'circle-stroke-width': 1,
  },
});
```

Here we've used a `filter` with the `!` operator, which is a Boolean NOT operator: this layer only shows points that do NOT have a `point_count` property.

Amazing!  By adding clusters to your map, you've reduced visual clutter,
and you've actually added more information in the process - now users
can see exactly how many collisions are in a rough area, something that
would have been impossible to see with all the overlapping points
before.

### Step 4d: visual differentiation

We could still do more to help users pick out the most important data
points.  Remember those KSI collisions?  We can use `clusterProperties`
on our clustered data source to mark clusters that have _any_ KSI
collisions in them.  Once we do that, we can use this property together
with `case` expressions to dramatically increase the prominence of KSI
collisions.

To get to the next step, you have to define the `ksiAny` cluster
property, use it to style clustered circles and text labels, and use the
`ksi` property on unclustered points to style those.

Let's start with the `ksiAny` _cluster property_, which we'll add to our existing `collisionsClustered` source using the `clusterProperties` option:

```js
map.addSource('collisionsClustered', {
  type: 'geojson',
  data: collisions,
  cluster: true,
  clusterMaxZoom: ZOOM_LEVEL_3,
  clusterProperties: {
    ksiAny: ['any', ['get', 'ksi']],
  },
  clusterRadius: 30,
});
```

`any` is a Boolean OR operator.  This means that our clusters will have a new `ksiAny` property, which will be `true` if _any_ collision in that cluster is a KSI collision (and `false` otherwise).

Now we can use `ksiAny` in our cluster layers and `ksi` in our unclustered layer.  Here's our clustered circle layer:

```js
map.addLayer({
  id: 'collisionsClustered',
  source: 'collisionsClustered',
  type: 'circle',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  filter: ['has', 'point_count'],
  paint: {
    'circle-color': [
      'case',
      ['get', 'ksiAny'], COLOR_KSI_FILL,
      COLOR_COLLISION_FILL,
    ],
    'circle-opacity': [
      'interpolate',
      ['linear'],
      ['zoom'],
      ZOOM_LEVEL_1, 0,
      ZOOM_LEVEL_1 + 1, 1,
    ],
    'circle-radius': [
      '*',
      [
        'case',
        ['get', 'ksiAny'], 1.25,
        1,
      ],
      [
        'step',
        ['get', 'point_count'],
        8,
        10, 10,
        100, 14,
        1000, 16,
      ],
    ],
    'circle-stroke-color': [
      'case',
      ['get', 'ksiAny'], COLOR_KSI_STROKE,
      COLOR_COLLISION_STROKE,
    ],
    'circle-stroke-width': [
      'case',
      ['get', 'ksiAny'], 2,
      1,
  },
});
```

Looks like we need fill and stroke colors for KSI collisions:

```js
const COLOR_KSI_FILL = '#272727';
const COLOR_KSI_STROKE = '#fefefe';
```

Now we can also use `ksiAny` with these colors in our cluster text label layer:

```js
'text-color': [
  'case',
  ['get', 'ksiAny'], COLOR_KSI_STROKE,
  COLOR_COLLISION_STROKE,
],
```

Finally, we update the unclustered layer, using `ksi` instead of `ksiAny` to style unclustered KSI collisions:

```js
map.addLayer({
  id: 'collisionsUnclustered',
  source: 'collisionsClustered',
  type: 'circle',
  minzoom: ZOOM_LEVEL_1,
  maxzoom: ZOOM_LEVEL_3,
  filter: ['!', ['has', 'point_count']],
  paint: {
    'circle-color': [
      'case',
      ['get', 'ksi'], COLOR_KSI_FILL,
      COLOR_COLLISION_FILL,
    ],
    'circle-opacity': [
      'interpolate',
      ['linear'],
      ['zoom'],
      ZOOM_LEVEL_1, 0,
      ZOOM_LEVEL_1 + 1, 1,
    ],
    'circle-radius': [
      'case',
      ['get', 'ksi'], 6,
      4,
    ],
    'circle-stroke-color': [
      'case',
      ['get', 'ksi'], COLOR_KSI_STROKE,
      COLOR_COLLISION_STROKE,
    ],
    'circle-stroke-width': [
      'case',
      ['get', 'ksi'], 2,
      1,
    ],
  },
});
```

And here we are!  You've taken this map all the way from a blank canvas
to something that helps users see the big picture _and_ zoom in on
details, all while drawing attention to the most important points in the
dataset.

# Now What?

Is there more we could do?  Definitely!  Contextual popups, interactive
filters (e.g. collisions that involve cyclists, pedestrians, etc.),
additional datasets (e.g. traffic counts, road infrastructure projects,
etc.) - all these (and more!) could be added to unlock new possibilities
for understanding this dataset.

There's also lots we could do to improve accessibility.  The red text on
red circles is _definitely_ not WCAG AA compliant: it's too small, and
the contrast isn't high enough.  The map has no non-visual alternatives
(e.g. summary text), nor does it offer ways to navigate data by
keyboard.  (This is a non-trivial problem, of course!)

Finally: we're using a subset of the data.  Because of that, we can just
load it directly into the browser and play around - but what if we were
using the full 35-year dataset with 1.5 million data points, each of
which have several dozen properties?  To handle that much data, we need
to preprocess it: pre-generate vector tiles, pre-aggregate summary
statistics, pre-cluster point datasets.  By doing this, we can scale
these same techniques to work with millions (or even billions!) of data
points.
