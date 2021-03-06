# Threebox Documentation

<br>

## Background

Threebox works by adding a *Three.js* scene to *Mapbox GL*, creating a new *Mapbox GL* custom layer that implements [CustomLayerInterface](https://docs.mapbox.com/mapbox-gl-js/api/properties/#customlayerinterface). The custom layer API takes a fair amount of finessing to be useful, and Threebox tackles several hurdles to getting *Three.js* and *Mapbox GL* to work together. 

<br>

- - -

## Threebox

### Using Threebox

The instance of Threebox will be used normally across the full page, so it's recommended to be created at `window` scope to be used as a global variable, but this also will require to explicitly `dispose` the instance, otherwise it can produce memory leaks.

#### constructor

```js
var tb = new Threebox(map, mapboxGLContext [, options])
```

Sets up a threebox scene inside a [*Mapbox GL* custom layer's onAdd function](https://www.mapbox.com/mapbox-gl-js/api/#customlayerinterface), which provides both inputs for this method. Automatically synchronizes the camera movement and events between *Three.js* and *Mapbox GL* JS. 

| option | required | default | type   | purpose                                                                                  |
|-----------|----------|---------|--------|----------------------------------------------------------------------------------------------|
| `defaultLights`    | no       | false      | boolean | Whether to add some default lighting to the scene. If no lighting added, most objects in the scene will render as black |
| `realSunlight`    | no       | false      | boolean | It sets lights that simulate Sun position for the map center coords (`map.getCenter`) and user local datetime (`new Date()`). This sunlight can be updated through `tb.setSunlight` method. It calls internally to suncalc module. |
| `passiveRendering`     | no       | true   | boolean  | Color of line. Unlike other Threebox objects, this color will render on screen precisely as specified, regardless of scene lighting |
| `enableSelectingFeatures`     | no       | false   | boolean  | Enables the Mouseover and Selection of fill-extrusion features. This will fire the event `SelectedFeatureChange` |
| `enableSelectingObjects`     | no       | false   | boolean  | Enables the Mouseover and Selection of 3D objects. This will fire the event `SelectedChange`|
| `enableDraggingObjects`     | no       | false   | boolean  | Enables to the option to Drag a 3D object. This will fire the event `ObjectDragged` where `draggedAction = 'translate'`|
| `enableRotatingObjects`     | no       | false   | boolean  | Enables to the option to Drag a 3D object. This will fire the event `ObjectDragged` where `draggedAction = 'rotate'`|
| `enableToltips`     | no       | false   | boolean  | Enables the default tooltips on fill-extrusion features and 3D Objects`|

The setup will require to call recursively to `tb.update();` to render the Threebox scene. This 
[CustomLayerInterface#render](https://docs.mapbox.com/mapbox-gl-js/api/properties/#customlayerinterface#render)

Rerender the threebox scene. Fired in the custom layer's `render` function.
                                                              

The `mapboxGLContext` instance can be obtained in different ways. The most usual one is to get the context from the instance of the member [`onAdd(map, gl)`](https://docs.mapbox.com/mapbox-gl-js/api/properties/#customlayerinterface#onadd), but that implies that it's created in every call to the method [`addLayer(layer[, beforeId])`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map#addlayer)
```js
map.addLayer({
	id: 'custom_layer',
	type: 'custom',
	renderingMode: '3d',
	onAdd: function (map, gl) {

		window.tb = new Threebox(
			map,
			gl, //get the context from Mapbox
			{ defaultLights: true }
		);
		...
		...
	},
	render: function (gl, matrix) {
		tb.update(); //update Threebox scene
	}
	
}
```
The second way is to the context from the canvas of the *Mapbox GL* map [`map.getCanvas().getContext('webgl')`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext). In this way the creation of the Threebox can be instantiated separately from the `addLayer` method.

```js
window.tb = new Threebox(
	map,
	map.getCanvas().getContext('webgl'), //get the context from the map canvas
	{ defaultLights: true }
);
```
<br>

- - -

### Loading a 3D model

One of the most powerful capabilities of Threebox is the option to load 3D models from external files in different formats (OBJ/MTL, GLTF/GLB, FBX, DAE are supported). 
Once the model is loaded and added to Threebox, the object is powered by default with some methods, interactions, events and animations. 

Once the object is loaded and added to Threebox instance, it can be **selectable**, **draggable** and **rotable** (over the z axis) 
with the mouse if Threebox instance properties `enableSelectingObjects`, `enableDraggingObjects` and 
`enableRotatingObjects` are set to `true`.

- To **drag** an object you have to select the object and then press **SHIFT** key and move the mouse.
- To **rotate** and object you have to select the object and then press **ALT** key and move the mouse. The object will always rotate over its defined center.

Any 3D object (including 3D extrusions created through fill-extrusion mapbox layers) will have a tooltip if the Threebox instance property `enableTooltips` is set to true.

Here below is the simplest sample to load a 3D model:

```html
<!doctype html>
<head>
	<title>Simplest sample of 3D Model loading</title>
	<script src="../dist/threebox.js" type="text/javascript"></script>
	<script src="https://api.mapbox.com/mapbox-gl-js/v.1.11.1/mapbox-gl.js"></script>
	<link href="https://api.mapbox.com/mapbox-gl-js/v.1.11.1/mapbox-gl.css" rel="stylesheet" />
	<style>
		body, html {
			width: 100%;
			height: 100%;
			margin: 0;
			background: black;
		}
		#map {
			width: 100%;
			height: 100%;
		}
	</style>
</head>
<body>
	<div id='map' class='map'></div>
	<script>
		mapboxgl.accessToken = 'Paste here your mapbox access token key';

		var origin = [-122.47920912, 37.716351775];
		var destination, line;
		var soldier;

		var map = new mapboxgl.Map({
			container: 'map',
			style: 'mapbox://styles/mapbox/outdoors-v11',
			center: origin,
			zoom: 18,
			pitch: 60,
			bearing: 0
		});

		map.on('style.load', function () {
			map.addLayer({
				id: 'custom_layer',
				type: 'custom',
				renderingMode: '3d',
				onAdd: function (map, mbxContext) {

					window.tb = new Threebox(
						map,
						mbxContext,
						{ defaultLights: true }
					);

					var options = {
						obj: '/3D/soldier/soldier.glb',
						type: 'gltf',
						scale: 1,
						units: 'meters',
						rotation: { x: 90, y: 0, z: 0 } //default rotation
					}

					tb.loadObj(options, function (model) {
						soldier = model.setCoords(origin);
						tb.add(soldier);
					})

				},
				render: function (gl, matrix) {
					tb.update();
				}
			});
		})
	</script>
</body>
```

<br>

- - -

### Threebox Methods

Here is the full list of all the methods exposed by Threebox. 
In all the samples below, the instance of Threebox will be always referred as `tb`.

#### add 
```js
tb.add(obj)
```
method to add an object to Threebox scene. It will add it to `tb.world.children` array.

<br>


#### clear 
```js
tb.clear([dispose])
```
This method removes any children from `tb.world`. if it receives `true` as a param, it will also call `obj.dispose` to dispose all the resources reserved by those objects.

<br>


#### defaultLights 
```js
tb.defaultLights()
```
This method creates the default illumination of the Threebox `scene`. It creates a [`THREE.AmbientLight`](https://threejs.org/docs/#api/en/lights/AmbientLight) and two [`THREE.DirectionalLight`](https://threejs.org/docs/#api/en/lights/DirectionalLight).
These lights can be overriden manually adding custom lights to the Threebox `scene`.

<br>

#### dispose
```js
tb.dispose() : Promise (async)
```
If Threebox is being used as a part of a web application, it's recommended to dispose explicitely the instance of Threebox whenever 
its instance is not going to be used anymore or before navigating to another page that does not include Threebox, otherwise it's 
likely that you will face **memory leaks** issues not only due to Threebox, but also due to the internal use of *Mapbox GL* and *Three.js* instances needed to manage the objects.<br>
<br>
To dispose completely all the resources and memory Threebox can acumulate, including the internal resources from *Three.js* and *Mapbox GL*, it's needed to invoke the `dispose` method. <br>
<br>
This method will go through the scene created to dispose every object, geometry, material and texture in *Three.js*, then it will dispose all the resources from *Mapbox GL*, including the `WebGLRenderingContext` and itselft the Threebox instance.<br>
<br>
After calling to this method, Threebox and *Mapbox GL* map instances will be fully disposed so it's only recommended before navigating to other pages. 

<br>

#### findParent3DObject 
```js
tb.findParent3DObject(mesh) : Object3D
```
This method finds the parent Object3D in the Threebox scene by a mesh. This method is used in combination with `tb.queryRenderedFeatures` that returns an Array of objects, most of them Meshes where the [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) has interesected.

<br>


#### getFeatureCenter 
```js
tb.getFeatureCenter(feature, model, level): lnglat
```
Calculate the center of a feature geometry coordinates, including the altitude (in meters) for a given [*GeoJson*](https://geojson.org/) feature that can include or not a 3D model loaded and for a given level.
This method calls internally to `tb.getObjectHeightOnFloor` and can be used for both a Poligon feature for a Fill Extrusion or a Point feature for a 3D model.

<br>


#### getObjectHeightOnFloor 
```js
tb.getObjectHeightOnFloor(feature, obj, level) : number
```
Calculate the altitude (in meters) for a given [*GeoJson*](https://geojson.org/) feature that can include or not a 3D model loaded and for a given level.
This method can be used for both a Poligon feature for a Fill Extrusion or a Point feature for a 3D model.

<br>


#### loadObj
```js
tb.loadObj(options, callback(obj));
```

This method loads a 3D model in different formats from its respective files. 
Note that unlike all the other object classes, this is asynchronous, and returns the object as an argument of the callback function. 
Internally, uses [`THREE.OBJLoader`](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/OBJLoader.js), [`THREE.FBXLoader`](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/FBXLoader.js), [`THREE.GLTFLoader`](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/GLTFLoader.js) or [`THREE.ColladaLoader`](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/ColladaLoader.js) respectively to fetch the  assets to each 3D format. [`THREE.FBXLoader`](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/FBXLoader.js) also dependes on [Zlib](https://github.com/imaya/zlib.js) to open compressed files which this format is based on.

*[jscastro]* **IMPORTANT**: There are breaking changes in this release regarding the attributes below comparing to [@peterqliu original Threebox](https://github.com/peterqliu/threebox/). 

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|----------|
| `type`  | yes       | "mtl"       | string (`"mtl"`, `"gltf"`, `"fbx"`, `"dae"`) | **BREAKING CHANGE**: Type is now required |
| `obj`  | yes       | NA       | string | **BREAKING CHANGE**: URL path to asset's .obj, .glb, .gltf, .fbx, .dae file. |
| `bin`  | no       | NA       | string | URL path to asset's .bin or .mtl files needed for GLTF or OBJ models respectively|
| `units`    | no       | scene      | string (`"scene"` or `"meters"`) | "meters" is recommended for precision. Units with which to interpret the object's vertices. If meters, Threebox will also rescale the object with changes in latitude, to appear to scale with objects and geography nearby.|
| `rotation`     | no       | 0   | number or {x, y, z}  | Rotation of the object along the three axes, to align it to desired orientation before future rotations. Note that future rotations apply atop this transformation, and do not overwrite it. `rotate` attribute must be provided in number or per axis ((i.e. for an object rotated 90 degrees over the x axis `rotation: {x: 90, y: 0, z: 0}`|
| `scale`     | no       | 1   | number or {x, y, z}  | Scale of the object along the three axes, to size it appropriately before future transformations. Note that future scaling applies atop this transformation, rather than overwriting it. `scale` attribute must be provided in number or per axis ((i.e. for an object transformed to 3 times higher than it's default size  `scale: {x: 1, y: 1, z: 3}`|
| `anchor`     | no       | `bottom-left`   | string ()  | This param will position the pivotal center of the 3D models to the coords it's positioned. This could have the following values `top`, `bottom`, `left`, `right`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Default value is `bottom-left`. `auto` value will do nothing, so the model will use the anchor defined in the model, whatever it is. |
| `adjustment`     | no       | 1   | {x, y, z}  | 3D models are often not centered in their axes so the object positions and rotates wrongly. `adjustment` param must be provided in units per axis (i.e. `adjustment: {x: 0.5, y: 0.5, z: 0}`), so the model will correct the center position of the object |
| `normalize`     | no       | 1   | bool  | This param allows to normalize specular values from some 3D models |
| `feature`     | no       | 1   | [*GeoJson*](https://geojson.org/) feature  | [*GeoJson*](https://geojson.org/) feature instance. `properties` object of the *GeoJson* standard feature could be used to store relavant data to load and paint many different objects such as camera position, zoom, pitch or bearing, apart from the attributes already usually used by [*Mapbox GL* examples](https://docs.mapbox.com/mapbox-gl-js/examples/) such as `height`, `base_height`, `color`|
| `callback`     | yes       | NA   | function  | A function to run after the object loads. The first argument will be the successfully loaded object, and this is normally used to finish the configuration of the model and add it to Threebox scene through `tb.add()` method. 


```js
map.addLayer({
	id: 'custom_layer',
	type: 'custom',
	renderingMode: '3d',
	onAdd: function (map, gl) {

		window.tb = new Threebox(
			map,
			gl,
			{ defaultLights: true }
		);

		var options = {
			obj: '/3D/soldier/soldier.glb', //the model url, relative path to the page 
			type: 'gltf', //type enum, glb format is
			scale: 20, //20x the original size
			units: 'meters', //everything will be converted to meters in setCoords method				
			rotation: { x: 90, y: 0, z: 0 }, //default rotation
			adjustment: { x: 0, y: 0, z: 0 }, // model center is displaced
			feature: geoJsonFeature // a valid GeoJson feature
		}

		tb.loadObj(options, function (model) {
			//this wil position the soldier at the GeoJson feature coordinates 
			soldier = model.setCoords(feature.geometry.coordinates);
			tb.add(soldier);
		})

	},
	render: function (gl, matrix) {
		tb.update();
	}
});
```

After the callback is initiated, the object returned will have the following events already 
available to listen that enable the UI to behave and react to those. 
You can add these `addEventListener` lines below to `tb.loadObj`:

```js
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('SelectedChange', onSelectedChange, false);
		soldier.addEventListener('Wireframed', onWireframed, false);
		soldier.addEventListener('IsPlayingChanged', onIsPlayingChanged, false);
		soldier.addEventListener('ObjectDragged', onDraggedObject, false);
		soldier.addEventListener('ObjectMouseOver', onObjectMouseOver, false);
		soldier.addEventListener('ObjectMouseOut', onObjectMouseOut, false);

		tb.add(soldier);
	})
```

In this way you'll be able to manage in you UI through a function once these events are fired. 
See below an example for `onSelectedChange` to use the method [`map.flyTo(options[, eventData])`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map#flyto) from *Mapbox GL*:

```js
// method to activate/deactivate a UI button.
// this example uses jQuery 
// this example requires a GeoJson feature included in the options of the tb.loadObj(options) 
function onSelectedChange(e) {
	let selected = e.detail.selected; //we get if the object is selected after the event
	$('#deleteButton')[0].disabled = !selected; //we find the delete button with jquery

	//if selected
	if (selected) {
		selectedObject = e.detail; //
		//we fly smoothly to the object selected
		map.flyTo({
			center: selectedObject.userData.feature.properties.camera,
			zoom: selectedObject.userData.feature.properties.zoom,
			pitch: selectedObject.userData.feature.properties.pitch,
			bearing: selectedObject.userData.feature.properties.bearing
		});
	}
	tb.update();
	map.repaint = true;
}
```


##### 3D Formats and MIME types 
Most of the popular 3D formats extensions (.glb, .gltf, .fbx, .dae, ...) are not standard [MIME types](https://www.iana.org/assignments/media-types/media-types.xhtml), so you will need to configure your web server engine to accept this extensions, otherwise you'll receive different HTTP errors downloading them. 
<br>
If you are using **IIS** server from an *ASP.Net* application, add the xml lines below in the `</system.webServer>` node of your *web.config* file:
```xml
<system.webServer>
	  ...
	  <staticContent>
		  <remove fileExtension=".mtl" />
		  <mimeMap fileExtension=".mtl" mimeType="model/mtl" />
		  <remove fileExtension=".obj" />
		  <mimeMap fileExtension=".obj" mimeType="model/obj" />
		  <remove fileExtension=".glb" />
		  <mimeMap fileExtension=".glb" mimeType="model/gltf-binary" />
		  <remove fileExtension=".gltf" />
		  <mimeMap fileExtension=".gltf" mimeType="model/gltf+json" />
		  <remove fileExtension=".fbx" />
		  <mimeMap fileExtension=".fbx" mimeType="application/octet-stream" />
	  </staticContent>
</system.webServer>
```
If you are using an **nginx** server, add the following lines to the *nginx.conf* file in the `http` object:
```
http {
	include /etc/nginx/mime.types;
	types {
		model/mtl mtl;
		model/obj obj;
		model/gltf+json gltf;
		model/gltf-binary glb;
		application/octet-stream fbx;
	}
	...
}
```
If you are using an **Apache** server, add the following lines to the *mime.types* file:
```
model/mtl mtl
model/obj obj
model/gltf+json gltf
model/gltf-binary glb
application/octet-stream fbx
```

<br>

#### memory 
```js
tb.memory() : Object
```
This will return the member `memory` from [`THREE.WebGLRenderer.info`](https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info)

<br>

#### programs 
```js
tb.programs() : int
```
This will return the lenght of the `programs` member from [`THREE.WebGLRenderer.info`](https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info)

<br>

#### projectToWorld 
```js
tb.projectToWorld(lnglat) : THREE.Vector3
```
Calculate the corresponding [`THREE.Vector3`](https://threejs.org/docs/#api/en/math/Vector3) for a given `lnglat`. It's inverse method is `tb.unprojectFromWorld`.

<br>

#### queryRenderedFeatures 
```js
tb.queryRenderedFeatures(point) : Array
```
This methods calculate objects intersecting the picking ray using [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) and returns an Array of the Threebox objects in the scene ordered by distance from closer to farther away.

Takes an input of `{x: number, y: number}` as an object with values representing screen coordinates (as returned by *Mapbox GL*  mouse events as `e.point`). 

<br>

#### realSunlight 
```js
tb.realSunlight()
```
This method creates the an illumination that simulates Sun light for the Threebox `scene`. It creates a [`THREE.HemisphereLight`](https://threejs.org/docs/index.html#api/en/lights/HemisphereLight) and one [`THREE.DirectionalLight`](https://threejs.org/docs/#api/en/lights/DirectionalLight) 
that is positioned based on `suncalc.js.` module which calculates the sun position for a given date, time, lng, lat combination. It calls internally to `tb.setSunlight` with `map.getCenter` and `new Date()` as values.
These lights can be overriden manually adding custom lights to the Threebox `scene`.

<br>

#### remove 
```js
tb.remove(obj)
```
method to remove an object from Threebox scene and the `tb.world.children` array.

<br>

#### setLayerHeigthProperty 
```js
tb.setLayerHeigthProperty(layerId, level) 
```
method to set the height of all the objects in a level. 
This method only works if the objects have a [*GeoJson*](https://geojson.org/) feature, and a `level` attribute among its properties.

<br>

#### setLayerZoomRange 
```js
tb.setLayerZoomRange(layer3d, minZoomLayer, maxZoomLayer)
```
Custom Layers don't work on minzoom and maxzoom attributes, and if the layer is including labels they don't hide either on minzoom

<br>


#### setLabelZoomRange 
```js
tb.setLabelZoomRange(minzoom, maxzoom)  
```
method set the CSS2DObjects zoom range and hide them at the same time the layer is

<br>


#### setLayoutProperty 
```js
tb.setLayoutProperty(layerId, name, value)
```
This method to replicates the behaviour of [`map.setLayoutProperty`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map#setlayoutproperty) when custom layers are affected but it can used for any layer type. 

<br>


#### setStyle 
```js
tb.setStyle(styleId[, options])
```
This method to replicates the behaviour of [`map.setStyle`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map#setstyle) to remove the 3D objects from `tb.world`.
It's a direct passthrough to `map.setStyle` but it also calls internally `tb.clear(true)` to remove the children from `tb.world` and also to dispose all the resources reserved by those objects.

<br>

#### setSunlight 
```js
tb.setSunlight(newDate = new Date(), coords)
```
This method updates Sun light position based on `suncalc.js.` module which calculates the sun position for a given date, time, lng, lat combination. It calls internally to `tb.setSunlight` with `map.getCenter` and `new Date()` as values.

<br>

#### toggleLayer 
```js
tb.toggleLayer(layerId, visible) 
```
This method to toggles any layer visibility but it's specifically designed for custom layers. 
If you want to avoid differentiating between Mapbox layers (including custom layers) this method replaces the call to [`map.setLayoutProperty(layerId, 'visibility', visible)`](https://docs.mapbox.com/mapbox-gl-js/api/map/#map#setlayoutproperty)

<br>


#### update 
```js
tb.update()
```
With `tb.loadObj(options, callback(obj))` this is probably the most important method in Threebox 
as it's responsible of invoking the [`THREE.WebGLRenderer.render(scene, camera)`](https://threejs.org/docs/#api/en/renderers/WebGLRenderer.render)
 method.

<br>

#### unprojectFromWorld 
```js
tb.unprojectFromWorld(Vector3): lnglat
```
Calculate the corresponding `lnglat` for a given [`THREE.Vector3`](https://threejs.org/docs/#api/en/math/Vector3). It's inverse method is `tb.projectToWorld`.

<br>

#### versions 
```js
tb.version() : string
```
This will return the version of Threebox

<br>

- - -

## Objects

Threebox offers convenience functions to construct meshes of various *Three.js* meshes, as well as 3D models. 
Under the hood, they invoke a subclass of [THREE.Object3D](https://threejs.org/docs/#api/en/core/Object3D). 

Objects in Threebox fall under two broad varieties. *Static objects* don't move or change once they're placed, 
and used usually to display background or geographical features. They may have complex internal geometry, which 
are expressed primarily in lnglat coordinates. 

In contrast, *dynamic objects* can move around the map, positioned by a single lnglat point. 
Their internal geometries are produced mainly in local scene units, whether through external obj files, or these convenience 
methods below.

<br>

### Static objects

#### Line 

```js
tb.line(options);
```

Adds a line to the map, in full 3D space. Color renders independently of scene lighting. Internally, calls a [custom line shader](https://threejs.org/examples/?q=line#webgl_lines_fat).


| option | required | default | type   | purpose                                                                                  |
|-----------|----------|---------|--------|----------------------------------------------------------------------------------------------|
| `geometry`    | yes       | NA      | lineGeometry | Array of lnglat coordinates to draw the line |
| `color`     | no       | black   | color  | Color of line. Unlike other Threebox objects, this color will render on screen precisely as specified, regardless of scene lighting |
| `width`     | no       | 1   | number  | Line width. Unlike other Threebox objects, this width is in units of display pixels, rather than meters or scene units. |
| `opacity`     | no       | 1   | Number  | Line opacity |                                                                       


<br>

### Dynamic objects

#### Label

```js
tb.label(options);
```

Creates a new HTML Label object that can be positioned through `obj.setCoords(coords)` and then added to Threebox scene through `tb.add(obj)`.
Internally this method uses a `CSS2DObject` rendered by [`THREE.CSS2DRenderer`](https://threejs.org/docs/#examples/en/renderers/CSS2DRenderer) to create an instance of `THREE.CSS2DObject` that will be associated to the `obj.label` property.

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|-------|
| `htmlElement`    | yes       | null      | htmlElement | HTMLElement that will be rendered as a `CSS2DObject` |
| `cssClass`    | no       | " label3D"      | string | CssClass that will be aggregated to manage the styles of the label object. |
| `alwaysVisible`  | no       | false       | number | Number of width and height segments. The higher the number, the smoother the sphere. |
| `topMargin`     | no       | -0.5   | int  | If `topMargin` is defined in number, it will be added to it's vertical position in units, where 1 is the object height. By default a label will be positioned in the vertical middle of the object (`topMargin: -0.5`)|
| `feature`     | no       | null   | [*GeoJson*](https://geojson.org/) feature  | [*GeoJson*](https://geojson.org/) feature to assign to the tooltip. It'll be used for dynamic positioning |                                                                            

<br>

#### Sphere

```js
tb.sphere(options);
```

Add a sphere to the map. Internally, calls `THREE.Mesh` with a `THREE.SphereGeometry`, and also to `Object3D(options)` to convert it in a dynamic object.

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|-------|
| `radius`    | no       | 50      | number | Radius of sphere. |
| `units`    | no       | `scene`      | string ("scene" or "meters") | Units with which to interpret the object's vertices. If meters, Threebox will also rescale the object with changes in latitude, to appear to scale with objects and geography nearby.|
| `sides`  | no       | 8       | number | Number of width and height segments. The higher the number, the smoother the sphere. |
| `color`     | no       | black   | color  | Color of sphere.                                                                             
| `material`     | no       | MeshLambertMaterial   | threeMaterial  | [THREE material](https://github.com/mrdoob/three.js/tree/master/src/materials) to use. Can be invoked with a text string, or a predefined material object via THREE itself.|   
| `anchor`     | no       | `bottom-left`   | string | This param will position the pivotal center of the 3D models to the coords it's positioned. This could have the following values `top`, `bottom`, `left`, `right`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Default value is `bottom-left` |
| `adjustment`     | no       | 1   | {x, y, z}  | For geometries the center is by default {0,0,0} position, this is the point to be used for location and for rotation. For perfect positioning and heigth from floor calculations this could be redefined in normalized units, `adjustment` param must be provided in units per axis (i.e. `adjustment: {x: -0.5, y: -0.5, z: 0}` , so the model will correct the center position of the object minus half of the x axis length and minus half of the y axis length ). If you position a cube created throuhg this method with by default center in a concrete `lnglat`on 0 height, half of the cube will be below the ground map level and the object will position at it's `{x,y}` center, so you can define `adjustment: { x: -0.5, y: -0.5, z: 0.5 }` to change the center to the bottom-left corner and that corner will be exactly in the `lnglat` position at the ground level. |

<br>

#### Tooltip

```js
tb.tooltip(options);
```

Creates a new browser-like Tooltip object that can be positioned through `obj.setCoords(coords)` and then added to Threebox scene through `tb.add(obj)`.
Internally this method uses a `CSS2DObject` rendered by [`THREE.CSS2DRenderer`](https://threejs.org/docs/#examples/en/renderers/CSS2DRenderer) to create an instance of `THREE.CSS2DObject` that will be associated to the `obj.tooltip` property.

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|-------|
| `text`    | yes       | ""      | string | String that will be used to rendered as a `CSS2DObject` |
| `cssClass`    | no       | "toolTip text-xs"      | string | CssClass that will be aggregated to manage the styles of the label object. |
| `mapboxStyle`     | no       | false   | int  | If `mapboxStyle` is true, it applies the same styles the *Mapbox GL* popups. |                                                                            
| `topMargin`     | no       | 0   | int  | If `topMargin` is defined in number, it will be added to it's vertical position in units, where 1 is the object height. By default a label will be positioned on top of the object (`topMargin: 0`)|                                
| `feature`     | no       | null   | [*GeoJson*](https://geojson.org/) feature  | [*GeoJson*](https://geojson.org/) feature to assign to the tooltip. It'll be used for dynamic positioning |                                                                            

<br>

#### Tube

```js
tb.tube(options);
```

Extrude a tube along a specific lineGeometry, with an equilateral polygon as cross section. Internally uses a custom tube geometry generator, and also to `Object3D(options)` to convert it in a dynamic object.

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|----------|
| `geometry`    | yes       | NA      | lineGeometry | Line coordinates forming the tube backbone |
| `radius`    | no       | 20      | number | Radius of the tube cross section, or half of tube width.|
| `sides`  | no       | 8       | number | Number of facets along the tube. The higher, the more closely the tube will approximate a smooth cylinder. |
| `material`     | no       | MeshLambertMaterial   | threeMaterial  | [THREE material](https://github.com/mrdoob/three.js/tree/master/src/materials) to use. Can be invoked with a text string, or a predefined material object via THREE itself.|   
| `color`     | no       | black   | color  | Tube color. Ignored if `material` is a predefined `THREE.Material` object.  |
| `opacity`     | no       | 1   | Number  | Tube opacity |                                                                                                                                                   
| `anchor`     | no       | `bottom-left`   | string | This param will position the pivotal center of the 3D models to the coords it's positioned. This could have the following values `top`, `bottom`, `left`, `right`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Default value is `bottom-left` |
| `adjustment`     | no       | 1   | {x, y, z}  | For geometries the center is by default {0,0,0} position, this is the point to be used for location and for rotation. For perfect positioning and heigth from floor calculations this could be redefined in normalized units, `adjustment` param must be provided in units per axis (i.e. `adjustment: {x: -0.5, y: -0.5, z: 0}` , so the model will correct the center position of the object minus half of the x axis length and minus half of the y axis length ). If you position a cube created throuhg this method with by default center in a concrete `lnglat`on 0 height, half of the cube will be below the ground map level and the object will position at it's `{x,y}` center, so you can define `adjustment: { x: -0.5, y: -0.5, z: 0.5 }` to change the center to the bottom-left corner and that corner will be exactly in the `lnglat` position at the ground level. |

<br>

#### Object3D

```js
tb.Object3D(options)
```

Add any geometry as [`THREE.Object3D`](https://threejs.org/docs/#api/en/core/Object3D) or [`THREE.Mesh`](https://threejs.org/docs/index.html#api/en/objects/Mesh) instantiated elsewhere in *Three.js*, to empower it with Threebox methods below. Unnecessary for 3d models instantiated with `tb.loadObj` above.

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|------------|
| `obj`    | yes       | null      | [`THREE.Mesh`](https://threejs.org/docs/index.html#api/en/objects/Mesh) | Object to be enriched with this method adding new attributes. |
| `units`    | no       | `scene`      | string ("scene" or "meters") | Units with which to interpret the object's vertices. If meters, Threebox will also rescale the object with changes in latitude, to appear to scale with objects and geography nearby.|
| `anchor`     | no       | `bottom-left`   | string | This param will position the pivotal center of the 3D models to the coords it's positioned. This could have the following values `top`, `bottom`, `left`, `right`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Default value is `bottom-left` |
| `adjustment`     | no       | 1   | {x, y, z}  | For geometries the center is by default {0,0,0} position, this is the point to be used for location and for rotation. For perfect positioning and heigth from floor calculations this could be redefined in normalized units, `adjustment` param must be provided in units per axis (i.e. `adjustment: {x: -0.5, y: -0.5, z: 0}` , so the model will correct the center position of the object minus half of the x axis length and minus half of the y axis length ). If you position a cube created throuhg this method with by default center in a concrete `lnglat`on 0 height, half of the cube will be below the ground map level and the object will position at it's `{x,y}` center, so you can define `adjustment: { x: -0.5, y: -0.5, z: 0.5 }` to change the center to the bottom-left corner and that corner will be exactly in the `lnglat` position at the ground level. |

This method enriches the Object in the same way is done at 3D Models through `tb.loadObj`.

<br>

---


### Object methods

In all the samples below, the instance of the Threebox object will be always referred as `obj`

<br>

#### addLabel
```js
obj.addLabel(HTMLElement [, visible, mapboxSyle = false, center = obj.anchor])

```
It uses the DOM [HTMLElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) received to paint it on screen in a relative position to the object that contains it. 
If `visible` is true, the label will be always visible, otherwise by default its value is false and it's regular behavior is only to be shown on MouseOver.
`center` defines the object's center of position and rotation that in 3D objects is defined through `options.adjustment` param. As the label is 
calculated based on the center of the object, this value will change the position of the object.
Its position is always relative to the object that contains it and rerendered whenever that label is visible.
Internally this method uses a `CSS2DObject` rendered by [`THREE.CSS2DRenderer`](https://threejs.org/docs/#examples/en/renderers/CSS2DRenderer) to create an instance of `THREE.CSS2DObject` that will be associated to the `obj.label` property.

*TODO: In next versions of Threebox, this object position will be configurable. In this versi�n it's positioned at the top of the object.*

<br>

#### addTooltip
```js
obj.addTooltip(tooltipText [, mapboxSyle = false, center = obj.anchor])
```
This method creates a browser-like tooltip for the object using the tooltipText. 
If `mapboxStyle` is true, it applies the same styles the *Mapbox GL* popups.
`center` defines the object center of position and rotation that in 3D objects is defined through `options.adjustment` param. As the tooltip is 
calculated based on the center of the object, this value will change the position of the object.
Its position is always relative to the object that contains it and rerendered whenever that label is visible.

Internally this method uses a `CSS2DObject` rendered by [`THREE.CSS2DRenderer`](https://threejs.org/docs/#examples/en/renderers/CSS2DRenderer) to create an instance of `THREE.CSS2DObject` that will be associated to the `obj.label` property.

*TODO: In next versions of Threebox, this object position will be configurable. In this versi�n it's positioned at the center of the object.*

<br>

#### drawBoundingBox
```js
obj.drawBoundingBox
```
This method creates two bounding boxes using [`THREE.Box3Helper`](https://threejs.org/docs/#api/en/helpers/BoxHelper)
The first bounding box will be assigned to `obj.boundingBox` property and the second will be assigned to `obj.boundingBoxShadow`.  

<br>

#### drawLabelHTML
```js
obj.drawLabelHTML(HTMLElement [, visible = false, center = obj.anchor])

```
It uses the DOM [HTMLElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) received to paint it on screen in a relative position to the object that contains it. 
If `visible` is true, the label will be always visible, otherwise by default its value is false and it's regular behavior is only to be shown on MouseOver.
`center` defines the object center of position and rotation that in 3D objects is defined through `options.adjustment` param. As the label is 
calculated based on the center of the object, this value will change the position of the object.
Its position is always relative to the object that contains it and rerendered whenever that label is visible.
Internally this method uses a `CSS2DObject` rendered by [`THREE.CSS2DRenderer`](https://threejs.org/docs/#examples/en/renderers/CSS2DRenderer) to create an instance of `THREE.CSS2DObject` that will be associated to the `obj.label` property.




#### set
```js
obj.set(options)
```
Broad method to update object's position, rotation, and scale in only one call. Internally it calls to `obj._setObject(options)` method. 
This method can also be used to animate an object if `options.duration` has a value.
Check out the Threebox Types section below for details. 

**options object**

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|------------|
| `coords`    | no       | NA      | `lnglat` | Position to which to move the object |
| `rotation`    | no       | NA      | `rotationTransform` | Rotation(s) to set the object, in units of degrees |
| `scale`    | no       | NA      | `scaleTransform` | Scale(s) to set the object, where 1 is the default scale |
| `duration`    | no       | 1000      | number | Duration of the animation, in milliseconds to complete the values specified in the other properties `scale`, `rotation` and `coords`. If 0 or undefined it will apply the values to the object directly with no animation. |

<br>

#### setAnchor
```js
obj.setAnchor(anchor)
```
Sets the positional and pivotal anchor automatically from string param. Calculates dynamically the positional and pivotal anchor. 
`anchor` is a string value that could have the following values: `top`, `bottom`, `left`, `right`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. 
Default value is `bottom-left` for precison on positioning 

<br>

#### setBoundingBoxShadowFloor
```js
obj.setBoundingBoxShadowFloor()
```
This method is called from `obj.setCoords` every time an object receives new coords to position `obj.boundingBoxShadow` at the height of the floor. 
So in this way if an object changes it's height the shadow box always projects over the floor and it's easier to visualize, position and rotate dragging it.

<br>

#### setCoords
```js
obj.setCoords(lnglat)
```
Positions the object at the defined `lnglat` coordinates, and resizes it appropriately if it was instantiated with `units: "meters"`. 
Can be called before adding object to the map.

<br>

#### setRotation
```js
obj.setRotation(xyz)
```

Rotates the object over its defined center in the 3 axes, the value must be provided in degrees and could be a number (it will apply the rotation to the 3 axes) or as an {x, y, z} object. 
This rotation is applied on top of the rotation provided through loadObj(options).

<br>

#### setRotationAxis
```js
obj.setRotationAxis(xyz)
```

Rotates the object over one of its bottom corners on z axis, the value must be provided in degrees and could be a number (it will apply the rotation to the 3 axes) or as an {x, y, z} object. 
This rotation is applied on top of the rotation provided through loadObj(options).

<br>

#### setTranslate
```js
obj.setTranslate(lnglat)
```
Movesthe object from it's current position adding the `lnglat` coordinates recibed. Don't confuse this method with `obj.setCoords`
This method must be called after adding object to the map.

<br>

### Object properties

In all the samples below, the instance of the Threebox object will be always referred as `obj`.

<br>

#### boundingBox

```js
obj.boundingBox : THREE.Box3Helper
```
This get/set property receives and return a [`THREE.Box3Helper`](https://threejs.org/docs/#api/en/helpers/BoxHelper) which contains the object in it's initial size. 
`boundingBox` represents is visible once the object is on MouseOver (yellow) or Selected (green).

By Threebox design `.boundingBox` is hidden for [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) even when it's visible for the camera.

*TODO: In next versions of Threebox, this object material will be configurable. In this versi�n still predefined in Objects.prototype*

<br>

#### boundingBoxShadow

```js
obj.boundingBoxShadow : THREE.Box3Helper
```
This get/set property receives and return a [`THREE.Box3Helper`](https://threejs.org/docs/#api/en/helpers/BoxHelper) which contains the object in it's initial size but 0 height and projected to the floor of the map independently of its heigh position, so it acts as a shadow of the shape. 
`boundingBoxShadow` represents is visible once the object is on MouseOver or Selected in black color.

By Threebox design `.boundingBoxShadow` is hidden for [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) even when it's visible for the camera.

*TODO: In next versions of Threebox, this object material will be configurable. In this versi�n still predefined in Objects.prototype*

<br>

#### visibility

```js
obj.visibility : boolean
```
This get/set property receives and return a boolean value to override the property `visible` of a  [`THREE.Object3D`](https://threejs.org/docs/#api/en/core/Object3D.visible), 
adding also the same visibility value for `obj.label` and `obj.tooltip`

By Threebox design `.boundingBoxShadow` is hidden for [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) even when it's visible for the camera.

*TODO: In next versions of Threebox, this object material will be configurable. In this versi�n still predefined in Objects.prototype*

<br>

#### wireframe

```js
obj.wireframe : boolean
```
This get/set property receives and return a boolean value to convert an [`THREE.Object3D`](https://threejs.org/docs/#api/en/core/Object3D.visible) in wireframes or texture it. 

By Threebox design whenever an object is converted to wireframes, it's also hidden for [`THREE.Raycaster`](https://threejs.org/docs/#api/en/core/Raycaster) even when it's visible for the camera.


<br>


---


### Object events

In all the samples below, the instance of the Threebox object will be always referred as `obj`

<br>

#### IsPlayingChanged

```js
obj.addEventListener('IsPlayingChanged', onIsPlayingChanged, false)
```

This event is fired once an object changes its animation playing status, it means it will be fired both when an object animation starts and stops to play.
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('IsPlayingChanged', onIsPlayingChanged, false);

		tb.add(soldier);
	})

	...
});
...
function onIsPlayingChanged(eventArgs) {
	if (e.detail.isPlaying) {
		//do something in the UI such as changing a button state
	}
	else {
		//do something in the UI such as changing a button state
	}
}
```

<br>

#### ObjectDragged

```js
obj.addEventListener('ObjectDragged', onDraggedObject, false)
```

This event is fired when an object changes is dragged and dropped in a different position, and only once when `map.once('mouseup'` and  `map.once('mouseout'`. 
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`, and the action made during the dragging that cound be `"rotate"` if the object has been rotated on its center axis or `"translate"` if the object has been moved to other position . 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('ObjectDragged', onDraggedObject, false);

		tb.add(soldier);
	})

	...
});
...
function onDraggedObject(eventArgs) {
	let draggedObject = e.detail.draggedObject; // the object dragged
	let draggedAction = e.detail.draggedAction; // the action during dragging

	//do something in the UI such as changing a button state or updating the new position and rotation
}
```

<br>


#### ObjectMouseOver

```js
obj.addEventListener('ObjectMouseOver', onObjectMouseOver, false)
```

This event is fired when an object is overed by the mouse pointer.  
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('ObjectMouseOver', onObjectMouseOver, false);

		tb.add(soldier);
	})

	...
});
...
function onObjectMouseOver(eventArgs) {
	//do something in the UI such as adding help or showing this object attributes
}
```

<br>


#### ObjectMouseOut

```js
obj.addEventListener('ObjectMouseOut', onObjectMouseOut, false)
```

This event is fired when the mouse pointer leaves an object that has been overed.   
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('ObjectMouseOut', onObjectMouseOut, false);

		tb.add(soldier);
	})

	...
});
...
function onObjectMouseOut(eventArgs) {
	//do something in the UI such as removing help
}
```

<br>

#### SelectedChange

```js
obj.addEventListener('SelectedChange', onSelectedChange, false)
```
This event is fired once an object changes its selection status, it means it will be fired both when an object is selected or unselected.
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('SelectedChange', onSelectedChange, false);

		tb.add(soldier);
	})

	...
});
...
function onSelectedChange(eventArgs) {
	let selectedObject = eventArgs.detail; //we get the object selected/unselected
	let selectedValue = selectedObject.selected; //we get if the object is selected after the event
}
```

<br>

#### SelectedFeature

```js
map.on('SelectedFeature', onSelectedFeature)
```
This event is fired by Threebox usign the same pattern of mapbox the `feature-state` `select` and `hover` 
**TODO**
once an object changes its selection status, it means it will be fired both when an object is selected or unselected.
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	'id': 'room-extrusion',
	'type': 'fill-extrusion',
	'source': 'floorplan',
	'paint': {
	// See the Mapbox Style Specification for details on data expressions.
	// https://docs.mapbox.com/mapbox-gl-js/style-spec/#expressions
 
	// Get the fill-extrusion-color from the source 'color' property.
	'fill-extrusion-color':
		[
			'case',
			['boolean', ['feature-state', 'select'], false],
			'#ffff00',
			['boolean', ['feature-state', 'hover'], false],
			'#0000ff',
			['get', 'color']
		],
 
	// Get fill-extrusion-height from the source 'height' property.
	'fill-extrusion-height': ['get', 'height'],
 
	// Get fill-extrusion-base from the source 'base_height' property.
	'fill-extrusion-base': ['get', 'base_height'],
 
	// Make extrusions slightly opaque for see through indoor walls.
	'fill-extrusion-opacity': 0.5
}
});
//selected extrusion feature event
map.on('SelectedFeature', onSelectedFeature);
...
function onSelectedFeature(eventArgs) {
	let selectedObject = eventArgs.detail; //we get the object selected/unselected
	let selectedValue = selectedObject.selected; //we get if the object is selected after the event
}
```

<br>

#### Wireframed

```js
obj.addEventListener('Wireframed', onWireframed, false)
```

This event is fired once an object is changes its wireframe status, it means it will be fired both when an object is wireframed or textured again.
The event can be listened at any time once the `tb.loadObj` callback method is being executed.
An instance of the object that changes is returned in `eventArgs.detail`. 

```js
map.addLayer({
	...
	tb.loadObj(options, function (model) {

		soldier = model.setCoords(origin);

		soldier.addEventListener('Wireframed', onWireframed, false);

		tb.add(soldier);
	})

	...
});
...
function onWireframed(eventArgs) {
	if (e.detail.wireframe) {
		//do something in the UI such as changing a button state
	}
	else {
		//do something in the UI such as changing a button state
	}
}
```

<br>

---


### Object animations

#### playDefault
```js
obj.playDefault(options)
```
Plays the default embedded animation of a loaded 3D model.

**options object**

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|------------|
| `duration`    | no       | 1000      | number | Duration of the animation, in milliseconds |


<br>

#### playAnimation
```js
obj.playAnimation(options)
```
Plays one of the embedded animations of a loaded 3D model. The animation index must be set in the attribute `options.animation`

**options object**

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|------------|
| `animation`    | yes       | NA      | number | Index of the animation in the 3D model. If you need to check whats the index of the animation you can get the full array using `obj.animations`.|
| `duration`    | no       | 1000      | number | Duration of the animation, in milliseconds |


<br>


#### followPath
```js
obj.followPath(options [, callback] )
```

Translate object along a specified path. Optional callback function to execute when animation finishes

| option | required | default | type   | description                                                                                  |
|-----------|----------|---------|--------|------------|
| `path`    | yes       | NA      | lineGeometry | Path for the object to follow |
| `duration`    | no       | 1000      | number | Duration to travel the path, in milliseconds |
| `trackHeading`    | no       | true      | boolean | Rotate the object so that it stays aligned with the direction of travel, throughout the animation |

<br>


#### stop
```js
obj.stop()
```

Stops all of object's current animations.

<br>


#### duplicate
```js
obj.duplicate()
```
Returns a clone of the object. Greatly improves performance when handling many identical objects, by reusing materials and geometries.

<br>

- - -

## Threebox types

<b>pointGeometry</b> `[longitude, latitude(, meters altitude)]`

An array of 2-3 numbers representing longitude, latitude, and optionally altitude (in meters). When altitude is omitted, it is assumed to be 0. When populating this from a [*GeoJson*](https://geojson.org/) Point, this array can be accessed at `point.geometry.coordinates`.  

While altitude is not standardized in the [*GeoJson*](https://geojson.org/) specification, Threebox will accept it as such to position objects along the z-axis.   

<br>

#### lineGeometry

`[pointGeometry, pointGeometry ... pointGeometry]`

An array of at least two lnglat's, forming a line. When populating this from a [*GeoJson*](https://geojson.org/) Linestring, this array can be accessed at `linestring.geometry.coordinates`. 

<br>

#### rotationTransform

`number` or `{x: number, y: number, z: number}`

Angle(s) in degrees to rotate object. Can be expressed as either an object or number. 

The object form takes three optional parameters along the three major axes: x is parallel to the equator, y parallel to longitudinal lines, and z perpendicular to the ground plane.

The number form rotates along the z axis, and equivalent to `{z: number}`.

<br>

#### scaleTransform

`number ` or `{x: number, y: number, z: number}`

Amount to scale the object, where 1 is the default size. Can be expressed as either an object or number.

The three axes are identical to those of `rotationTransform`. However, expressing as number form scales all three axes by that amount.

#### threeMaterial

`string` or instance of `THREE.Material()`

Denotes the material used for an object. This can usually be customized further with `color` and `opacity` parameters in the same 

Can be expressed as a string to the corresponding material type (e.g. `"MeshPhysicalMaterial"` for `THREE.MeshPhysicalMaterial()`), or a prebuilt THREE material directly.

<br>

- - -

## Using vanilla *Three.js* in Threebox

Threebox implements many small affordances to make mapping run in *Three.js* quickly and precisely on a global scale. Whenever possible, use threebox methods to add, change, manage, and remove elements of the scene. Otherwise, here are some best practices:

- Use `threebox.Object3D` to add custom objects to the scene
- If you must interact directly with the THREE scene, add all objects to `threebox.world`.
- `tb.projectToWorld` to convert lnglat to the corresponding `Vector3()`

## Performance considerations

- Use `obj.duplicate()` when adding many identical objects. If your object contains other objects not in the `obj.children` collection, then those objects need to be cloned too.`
