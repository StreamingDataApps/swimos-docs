---
title: Transit Tutorial
layout: page
---

The Transit Tutorial walks you step-by-step through creating a real-time city transit streaming application. The application involves four types of business entities: vehicles, agencies, states, and countries. These entities form a simple hierarchy where each vehicle falls under exactly one (of 67 possible) agencies. Each agency likewise falls under exactly one state, and each state falls under exactly one country.

We will be utilizing an API provided by Cubic Transportation System's Umo Mobility Platform to retrieve real-time transit information -- NextBus, https://retro.umoiq.com/.

### <a name="project-creation"></a>Project Creation

Let’s start by creating the root project folder. We are calling the directory `transit`. Create the `server` and `client` directories within `transit`, then go into `server`.
```console
$ mkdir transit
$ cd transit
$ mkdir server client
$ cd server
```

#### <a name="prerequisites"></a>Prerequisites

A JDK for <a href="https://www.oracle.com/technetwork/java/javase/downloads/index.html">Java 11+</a> must be installed in order to implement the backend application. Click <a href="https://www.oracle.com/technetwork/java/javase/downloads/index.html">here</a> for JDK installation instructions. In conjunction with this, make sure your `JAVA_HOME` environment variable is pointed to your Java installation location, and that your `PATH` includes `JAVA_HOME`. To test the client side and subscribe to streaming data, we’ll use <a href="https://nodejs.org/en/">NodeJS</a>. Click <a href="https://nodejs.org/en/">here</a> for NodeJS installation instructions.

#### <a name="gradle-setup"></a>Gradle Setup

We’ll be using Gradle to build the backend Java application. Click here for installation instructions. We’ll start by initializing gradle in the `server` folder:

```console
$ gradle init

Select type of project to generate:
  1: basic
  2: application
  3: library
  4: Gradle plugin
Enter selection (default: basic) [1..4] 1

Select build script DSL:
  1: Groovy
  2: Kotlin
Enter selection (default: Groovy) [1..2] 1

Generate build using new APIs and behavior (some features may change in the next minor release)? (default: no) [yes, no] yes
Project name (default: server): swim-transit
```

##### <a name="minimum-files"></a>Minimum files

Take note of the files that have been created for you:
```console
build.gradle	gradle		gradlew		gradlew.bat	settings.gradle
```

We’ll find settings.gradle contains one line of significance:
```groovy
rootProject.name="swim-transit"
```

We’ll create `gradle.properties` and add the following lines:
```groovy
application.version=1.0.0
swim.version=4.0.1
```

We’ll copy an existing build.gradle file from here:
https://github.com/swimos/transit/blob/master/server/build.gradle
[Determine how much of build.gradle is strictly necessary, and whether it is necessary to explain creating this by scratch]

The gradle wrapper, which we strongly recommend using, has been created automatically for us. 

### <a name="nodejs-client-setup"></a>NodeJS client setup

Leave the server directory and go back up to `transit` and then enter the `client` directory we created previously and invoke `npm init`:
```console
cd  ..
cd  client
```

Within `client` we’ll set up NodeJS by invoking `npm init` and accepting all the defaults:
```console
npm init
```

[reference swimos installs]

### <a name="project-organization"></a>Project organization

#### <a name="directory-structure"></a>Directory structure

From the root project directory, the directory structure will look like this:

<img src="#" alt="directory-tree-for-transit-01.png">

We’ll now add the initial source files for the backend application.  Navigating to the server directory under the root project directory, we’ll create additional directory structure:

```console
cd ..
cd server
mkdir -p src/main
cd src/main
mkdir -p java/swim/transit/agent
mkdir resources
```

#### <a name="creating-initial-files"></a>Creating initial files

We will be creating the following files:
```console
server/src/main/resources/server.recon
server/src/main/resources/agencies.csv
server/src/main/java/module-info.java
```

We’ll create `server.recon` under the `resources` directory.  First we specify kernel properties that determine runtime settings. Then, within the @fabric definition we specify the java Class that will serve as our application plane. A Plane defines a single layer (sub-application) of application to support multi-layer applications. We define @store to use the file system as a lightweight backing store, though in-memory remains the primary state store.

Next, we need to provide a CSV data file that includes columns for `id`, `state`, and `country` and refers to real transit vehicles that will will traffic via a public transit API. The file with agency information can be downloaded here:

<a href="https://raw.githubusercontent.com/swimos/transit/master/server/src/main/resources/agencies.csv">
  https://raw.githubusercontent.com/swimos/transit/master/server/src/main/resources/agencies.csv
</a>

Lastly, we need to configure the client-facing end-point. We do that using the @web directive where we set the port to `9001`. We tie the end-point to the fabric using the `space` property so it names the fabric’s key. We turn off websocket compression by setting `serverCompressionLevel` and `clientCompressionLevel` to 0.

```java
@kernel(class: 'swim.store.db.DbStoreKernel', optional: true)
@kernel(class: 'swim.reflect.ReflectKernel', optional: true)

transit: @fabric {
  @plane(class: "swim.transit.TransitPlane")
  @store {
    path: "/tmp/swim-transit/"
  }
}

@web(port: 9001) {
  space: "transit"
  @websocket {
    serverCompressionLevel: 0# -1 = default; 0 = off; 1-9 = deflate level
    clientCompressionLevel: 0# -1 = default; 0 = off; 1-9 = deflate level
  }
}
```

Next, we’ll move onto the `module-info.java` which specifies internal library dependencies, dependencies for our backend service, and the service implementation our service provides.

```java
open module swim.transit {
  requires swim.xml;
  requires transitive swim.api;
  requires swim.server;
  requires java.logging;

  exports swim.transit;
  exports swim.transit.model;

  provides swim.api.plane.Plane with swim.transit.TransitPlane;
}
```

### <a name="first-contact"></a>First Contact

We will be creating the following files:

```console
server/src/main/resources/server.recon
server/src/main/java/module-info.java
server/src/main/java/swim/transit/TransitPlane.java
server/src/main/java/swim/transit/agent/Vehicle.java
```

#### <a name="creating-the-app-plane"></a>Creating the App Plane

We’ll implement `TransitPlane` under `server/src/main/java/swim/transit/TransitPlane.java`. We begin by extending `TransitPlane` from the `AbstractPlane` base class, which will allow us to declare the application routes. The agent corresponding to each route is declared using the `@SwimAgent` annotation. The application routes are declared using the `@SwimRoute` annotation. The route itself is defined via the generic `AgentRoute` type.

```java
package swim.transit;

import java.util.logging.Logger;

import swim.api.SwimAgent;
import swim.api.SwimRoute;
import swim.api.agent.AgentRoute;
import swim.api.plane.AbstractPlane;
import swim.api.space.Space;
import swim.kernel.Kernel;
import swim.server.ServerLoader;
import swim.transit.agent.VehicleAgent;
import swim.structure.Value;

public class TransitPlane extends AbstractPlane {
  private static final Logger log = Logger.getLogger(TransitPlane.class.getName());
  @SwimAgent("vehicle")
  @SwimRoute("/vehicle/:country/:state/:agency/:id")
  AgentRoute<VehicleAgent> vehicleAgent;

  public static void main(String[] args) {
    final Kernel kernel = ServerLoader.loadServer();
    final Space space = kernel.getSpace("transit");

    kernel.start();
    log.info("Running TransitPlane...");

    space.command("/vehicle/US/CA/poseurs/dummy", "fake", Value.empty());

    kernel.run(); // blocks until termination
  }
}
```

#### <a name="creating-a-web-agent"></a>Creating a Web Agent

We will now create a barebones Web Agent for the vehicle entity. To do that, we’ll start by extending `AbstractAgent` and override the `didStart()` method:

```java
package swim.transit.agent;

import swim.api.agent.AbstractAgent;

import java.util.logging.Logger;

public class VehicleAgent extends AbstractAgent {

  @Override
  public void didStart() {
    log.info(()-> String.format("Started Agent: %s", nodeUri()));
  }

}
```

#### <a name="interacting-with-web-agent"></a>Interacting with Web Agent

Now we can run the application and confirm that the plane starts up and an agent is instantiated within the plane. From the server directory, run the following:
```console
./gradlew build
./gradlew run

#### <a name="handling-a-command"></a>Handling a Command

Web Agent data types are simple Java classes that provide a serialization object called a form. With annotations, the amount of specification is very minimal. We’ll take an initial pass at the Vehicle data type by specifying the @Tag on the class, and the @Kind tag on the Form member variable. We’ll start with just a few fields: an id to identify a particular vehicle, and then latitude and longitude to track its geolocation. We will serialize to the internal format of SwimOS with the `Form.forClass() method`. We’ll next add a mandatory default constructor and then a more functional one. Next, we’ll use the Fluent API pattern and define getters and setters, where the latter return the object reference to support function chaining. 

```java
package swim.transit.model;

import java.util.Objects;
import swim.structure.Form;
import swim.structure.Kind;
import swim.structure.Tag;
import swim.structure.Value;

@Tag("vehicle")
public class Vehicle {

  private String id = "";
  private float latitude = 0.0f;
  private float longitude = 0.0f;
  private int secsSinceReport = -1;

  public Vehicle() {
  }

  public Vehicle(String id, float latitude, float longitude, int speed, int secsSinceReport
) {
    this.id = id;
    this.latitude = latitude;
    this.longitude = longitude;
    this.speed = speed;
    this.secsSinceReport = secsSinceReport;
  }

  public String getId() {
    return id;
  }

  public Vehicle withId(String id) {
    return new Vehicle(id, latitude, longitude, speed, secsSinceReport);
  }

  public float getLatitude() {
    return latitude;
  }

  public Vehicle withLatitude(float latitude) {
    return new Vehicle(id, latitude, longitude, speed, secsSinceReport);
  }

  public float getLongitude() {
    return longitude;
  }

  public Vehicle withLongitude(float longitude) {
    return new Vehicle(id, latitude, longitude, speed, secsSinceReport);
  }

  public int getSpeed() {
    return speed;
  }

  public Vehicle withSpeed(int speed) {
    return new Vehicle(id, uri, latitude, longitude, speed, secsSinceReport);
  }

  public int getSecsSinceReport() {
    return secsSinceReport;
  }

  public Vehicle withSecsSinceReport(int secsSinceReport) {
    return new Vehicle(id, uri, latitude, longitude, speed, secsSinceReport);
  }


  @Override
  public boolean equals(Object other) {
    if (this == other) {
      return true;
    } else if (other instanceof Vehicle) {
      final Vehicle that = (Vehicle) other;
      return Float.compare(latitude, that.latitude) == 0
          && Float.compare(longitude, that.longitude) == 0
          && speed == that.speed
          && secsSinceReport == that.secsSinceReport
          && Objects.equals(id, that.id);
    }
    return false;
  }

  @Override
  public int hashCode() {
    return Objects.hash(id, latitude, longitude, speed, secsSinceReport);
  }

  @Override
  public String toString() {
    return "Vehicle{"
        + "id='" + id + '\''
        + ", latitude=" + latitude
        + ", longitude=" + longitude
        + ", speed=" + speed
        + ", secsSinceReport=" + secsSinceReport
        + '}';
  }

  public Value toValue() {
    return Form.forClass(Vehicle.class).mold(this).toValue();
  }

  @Kind
  private static Form<Vehicle> form;
}
```

With a data structure for vehicles, we will now utilize it in the Vehicle web agent to represent state and initialize the agent via a `CommandLane`. Let’s add the following imports for the Command Lane.

```java
import swim.api.SwimLane;
import swim.api.agent.AbstractAgent;
import swim.api.lane.CommandLane;
```

Let’s the add the following code before `didStart()` in `VehicleAgent.java`.

```java
  @SwimLane("addVehicle")
  public CommandLane<Vehicle> addVehicle = this.<Vehicle>commandLane().onCommand(this::onVehicle);

  private void onVehicle(Vehicle v) {
    final long time = System.currentTimeMillis() - (v.getSecsSinceReport() * 1000L);
    final int oldSpeed = vehicle.get() != null ? vehicle.get().getSpeed() : 0;
    this.vehicle.set(v);
    speeds.put(time, v.getSpeed());
    if (speeds.size() > 10) {
      speeds.drop(speeds.size() - 10);
    }
    if (lastReportedTime > 0) {
      final float acceleration = (float) ((v.getSpeed() - oldSpeed)) / (time - lastReportedTime) * 3600;
      accelerations.put(time, Math.round(acceleration));
      if (accelerations.size() > 10) {
        accelerations.drop(accelerations.size() - 10);
      }
    }
    lastReportedTime = time;
  }

  @Override
  public void didStart() {
    log.info(()-> String.format("Started Agent: %s", nodeUri()));
  }
```

Lastly, we’ll invoke addVehicle form the TransitPlane changing the command lane from `fake` to `addVehicle` so that the line below

```java
space.command("/vehicle/US/CA/poseurs/dummy", "fake", Value.empty());
```

becomes


```java
space.command("/vehicle/US/CA/poseurs/dummy", "addVehicle", Value.empty());
```

Let’s re-run our code to confirm that our command has been received and state has been stored:
```console
./gradlew build
./gradlew run
```

#### <a name="acting-on-state-changes"></a>Acting on state changes

We will now great some lanes that hold typed state so that we can process the old and new state as new state changes stream in. In the previous iteration, we sent a command that included Vehicle state as input. Let’s store that state now for use within the agent as well as any external observers that need to follow these state changes. We’ll create a SwimLane named vehicle that will specify the externally accessible route along with the underlying state field of type `ValueLane<Vehicle>` so that we can retrieve state via get() and set(). Instead of simply overwriting the `speed` field for Vehicle state, we will store the last 10 speeds indexed by timestamp and derive acceleration using the SwimOS Map counterpart, MapLane, which provides `get()` and `put()` accessors. We’ll apply the @SwimTransient tag to the speeds and acceleration to opt out of writing to disk since there is no long-term or fault tolerance benefit to tracking current acceleration.

```java
package swim.transit.agent;

import swim.api.SwimLane;
import swim.api.SwimTransient;
import swim.api.agent.AbstractAgent;
import swim.api.lane.CommandLane;
import swim.api.lane.MapLane;
import swim.api.lane.ValueLane;
import swim.transit.model.Vehicle;

import java.util.logging.Logger;

public class VehicleAgent extends AbstractAgent {
  private static final Logger log = Logger.getLogger(VehicleAgent.class.getName());
  private long lastReportedTime = 0L;

  @SwimLane("vehicle")
  public ValueLane<Vehicle> vehicle;

  @SwimTransient
  @SwimLane("speeds")
  public MapLane<Long, Integer> speeds;

  @SwimTransient
  @SwimLane("accelerations")
  public MapLane<Long, Integer> accelerations;

  @SwimLane("addVehicle")
  public CommandLane<Vehicle> addVehicle = this.<Vehicle>commandLane().onCommand(this::onVehicle);

  private void onVehicle(Vehicle v) {
    final long time = System.currentTimeMillis() - (v.getSecsSinceReport() * 1000L);
    final int oldSpeed = vehicle.get() != null ? vehicle.get().getSpeed() : 0;
    this.vehicle.set(v);
    speeds.put(time, v.getSpeed());
    log.info(() -> String.format("onVehicle – speed: %d", speed);
    if (speeds.size() > 10) {
      speeds.drop(speeds.size() - 10);
    }


    if (lastReportedTime > 0) {
      final float acceleration = (float) ((v.getSpeed() - oldSpeed)) / (time - lastReportedTime) * 3600;
      accelerations.put(time, Math.round(acceleration));
      if (accelerations.size() > 10) {
        accelerations.drop(accelerations.size() - 10);
      }
      log.info(() -> String.format("onVehicle – acceleration: %f", acceleration);
    }
    lastReportedTime = time;
  }

  @Override
  public void didStart() {
    log.info(()-> String.format("Started Agent: %s", nodeUri()));
  }
}
```

Let’s re-run our code to observe recordings of speed and derivations of acceleration:
```console
./gradlew build
./gradlew run
```

### <a name="implement-transit-domain-models"></a>Implement transit domain models

#### <a name="principle-model-definitions"></a>Principle model definitions

We will now complete the implemention of the domain models, starting with the three principle models `Agency`, `Route`, and `Vehicle`. Agency provides a logical view for a specific transit system.

```java
// imports
. . .

@Tag("agency")
public class Agency {

  private String id = "";
  private String state = "";
  private String country = "";
  private int index = 0;

  public Agency() {
  }

  public Agency(String id, String state, String country, int index) {
    this.id = id;
    this.state = state;
    this.country = country;
    this.index = index;
  }

  // getters
  public String getId() {
    return id;
  }

  public String getState() {
    return state;
  }

  public String getCountry() {
    return country;
  }

  public int getIndex() {
    return index;
  }

  public String getUri() {
    return "/agency/" + getCountry() + "/" + getState() + "/" + getId();
  }

  // serialize to SwimOS
  public Value toValue() {
    return form().mold(this).toValue();
  }

  // standard overrides of equals, hashCode, and toString
  . . .

  // definition of Form object to leverage serialization functionality
  @Kind
  private static Form<Agency> form;

  public static Form<Agency> form() {
    if (form == null) {
      form = Form.forClass(Agency.class);
    }
    return form;
  }
}
```

With `Route`, we just concerns ourselves with the `tag` and `title`. There is very little to it, so we’ll just include the full body:

```java
@Tag("route")
public class Route {

  private String tag = "";
  private String title = "";

  public Route() {
  }

  public Route(String tag, String title) {
    this.tag = tag;
    this.title = title;
  }

  public String getTag() {
    return tag;
  }

  public Route withTag(String tag) {
    return new Route(tag, title);
  }

  public String getTitle() {
    return title;
  }

  public Route withTitle(String title) {
    return new Route(tag, title);
  }

  @Kind
  private static Form<Route> form;
}
```

Lastly, we’ll now complete the model definition for `Vehicle`. Let’s add fields for:

```java
  private String uri = "";
  private String agency = "";
  private String routeTag = "";
  private String dirId = "";
  private int index = 0;
  private String heading = "";
  private String routeTitle = "";
```

And update the non-default constructor to accept the added fields:

```java
 public Vehicle(
    String id, String uri, String agency, String routeTag, String dirId,
    float latitude, float longitude, int speed, int secsSinceReport, int index,
    String heading, String routeTitle) {
```

Aside from this, it is simply of a matter of adding the corresponding getters and setters, and updating the standard overrides of `equals()`, `hashCode()`, and `toString()`.

#### <a name="supporting-model-definitions"></a>Supporting model definitions

Next we’ll round our model definitions with BoundingBox, Routes, and Vehicles. We’ll capture quad regions to capture min and max longitude and latitude with BoundingBox:

```java
// imports
. . .

@Tag("bounds")
public class BoundingBox {

  float minLng = Float.MAX_VALUE;
  float minLat = Float.MAX_VALUE;
  float maxLng = Float.MIN_VALUE;
  float maxLat = Float.MIN_VALUE;

  public BoundingBox() {
  }

  public BoundingBox(float minLng, float minLat, float maxLng, float maxLat) {
    this.minLat = minLat;
    this.minLng = minLng;
    this.maxLat = maxLat;
    this.maxLng = maxLng;
  }

  // getters and standard overrides
  . . .

  @Kind
  private static Form<BoundingBox> form;
}
```

We’ll represent a list of Routes and Vehicles with specific model definitions that let us add elements and serialize as a list and map, respectively. 

Here’s `Routes`, where we represent a list of routes:

```java
// imports
. . .

@Tag("Routes")
public class Routes {

  private final List<Route> routes = new ArrayList<Route>();

  public Routes() {
  }

  public List<Route> getRoutes() {
    return routes;
  }

  public void add(Route route) {
    routes.add(route);
  }

  @Kind
  private static Form<Routes> form;
}
```

And here’s Vehicles, where we represent a mapping of vehicle url to vehicle:

```java
// imports
. . .

@Tag("vehicles")
public class Vehicles {

  private final Map<String, Vehicle> vehicles = new HashMap<String, Vehicle>();

  public Vehicles() {
  }

  public Map<String, Vehicle> getVehicles() {
    return vehicles;
  }

  public void add(Vehicle vehicle) {
    vehicles.put(vehicle.getUri(), vehicle);
  }

  @Kind
  private static Form<Vehicles> form;
}
```


### <a name="implement-agents-for-transit-domain"></a>Implement agents for transit domain

We’ve already implemented VehicleAgent, so we’ll move onto the remaining agents for state, country, and agent.

#### <a name="implement-state-agent"></a>Implement StateAgent

Let’s implement StateAgent to manage the agencies and vehicles operating within the State. The basic lanes we’ll create maintain high-level statistics and element collections.

```java
 @SwimLane("count")
  public ValueLane<Value> count;

 @SwimTransient
  @SwimLane("agencyCount")
  public MapLane<Agency, Integer> agencyCount;

 @SwimTransient
  @SwimLane("vehicles")
  public MapLane<String, Vehicle> vehicles;

 @SwimLane("speed")
  public ValueLane<Float> speed;

  @SwimTransient
  @SwimLane("agencySpeed")
  public MapLane<Agency, Float> agencySpeed;
```

The more interesting lanes allow us to link up members agents for Agency and Vehicle. We’ll make use of SwimOS’s `JoinValueLane` reflect the vehicle count and average vehicle speed for each Agency. We’ll reflect the topology of agencies and vehicles by aggregating each agency and each agency’s vehicles. Then, we’ll link to an agency with respect to specific lanes with `addAgency()`, while making use of AbstractAgent’s context object to send commands to other Web Agents. We will materialize the links using the downlink command exposed on all join lane types, passing in an Agency data object as the key:

```java
joinAgencySpeed.downlink(agency).nodeUri(agency.getUri()).laneUri("speed").open();
```

Here is the relevant code:

```java
 @SwimLane("joinAgencyCount")
  public JoinValueLane<Agency, Integer> joinAgencyCount = this.<Agency, Integer>joinValueLane()
      .didUpdate(this::updateCounts);

  public void updateCounts(Agency agency, int newCount, int oldCount) {
    int vCounts = 0;
    final Iterator<Integer> it = joinAgencyCount.valueIterator();
    while (it.hasNext()) {
      final Integer next = it.next();
      vCounts += next;
    }

    final int maxCount = Integer.max(count.get().get("max").intValue(0), vCounts);
    count.set(Record.create(2).slot("current", vCounts).slot("max", maxCount));
    agencyCount.put(agency, newCount);
  }

 @SwimLane("joinStateSpeed")
  public JoinValueLane<Agency, Float> joinAgencySpeed = this.<Agency, Float>joinValueLane()
      .didUpdate(this::updateSpeeds);

  public void updateSpeeds(Agency agency, float newSpeed, float oldSpeed) {
    float vSpeeds = 0.0f;
    final Iterator<Float> it = joinAgencySpeed.valueIterator();
    while (it.hasNext()) {
      final Float next = it.next();
      vSpeeds += next;
    }
    if (joinAgencyCount.size() > 0) {
      speed.set(vSpeeds / joinAgencyCount.size());
    }
    agencySpeed.put(agency, newSpeed);
  }

 @SwimLane("joinAgencyVehicles")
  public JoinMapLane<Agency, String, Vehicle> joinAgencyVehicles = this.<Agency, String, Vehicle>joinMapLane()
      .didUpdate((String key, Vehicle newEntry, Vehicle oldEntry) -> vehicles.put(key, newEntry))
      .didRemove((String key, Vehicle vehicle) -> vehicles.remove(key));

  @SwimLane("addAgency")
  public CommandLane<Agency> agencyAdd = this.<Agency>commandLane().onCommand((Agency agency) -> {
    joinAgencyCount
        .downlink(agency).nodeUri(agency.getUri()).laneUri("count").open();

    joinAgencyVehicles
        .downlink(agency).nodeUri(agency.getUri()).laneUri("vehicles").open();

    joinAgencySpeed
        .downlink(agency).nodeUri(agency.getUri()).laneUri("speed").open();

    context.command(
        "/country/" + getProp("country").stringValue(),
        "addAgency",
        agency.toValue().unflattened().slot("stateUri", nodeUri().toString()));
  });
```

#### <a name="implement-country-agent"></a>Implement CountryAgent

CountryAgent will be nearly identical to StateAgent, except we’ll aggregate on the state level. `addAgency` represents the essential difference.


```java
  @SwimLane("addAgency")
  public CommandLane<Value> agencyAdd = this.<Value>commandLane()
    .onCommand((Value value) -> {
      states.put(value.get("state").stringValue(), value.get("state").stringValue());
      joinStateCount
        .downlink(value.get("state"))
        .nodeUri(Uri.parse(value.get("stateUri").stringValue()))
        .laneUri("count").open();

      joinStateSpeed
        .downlink(value.get("state"))
        .nodeUri(Uri.parse(value.get("stateUri").stringValue()))
        .laneUri("speed").open();

      final Agency agency = Agency.form().cast(value);
      agencies.put(agency.getUri(), agency);
  });

```

#### <a name="implement-agency-agent"></a>Implement AgencyAgent

##### <a name="Implement NextBusHttpAPI transit wrapper"></a>Implement NextBusHttpAPI transit wrapper

There are two public transit APIs we will connect to from UmoIQ (https://retro.umoiq.com/xmlFeedDocs/NextBusXMLFeed.pdf):
https://retro.umoiq.com/service/publicXMLFeed?command=routeList&a={agencyId}
https://retro.umoiq.com/service/publicXMLFeed?command=vehicleLocations&a={agencyId}&t=0

###### <a name="route-list"></a>routeList

The first end-point corresponds to the `routeList` command, and will return a route object with a `tag`, `title`, and `shortTitle`. We will make use of the tag and title fields. Passing in “sf-muni” as the agency would correspond to the following input and output.

INPUT:

```java
https://retro.umoiq.com/service/publicXMLFeed?command=routeList& a=sf-muni
```

OUTPUT:

```xml
<body>
<route tag="1" title="1 - California" shortTitle="1-Calif/>
<route tag="3" title="3 - Jackson" shortTitle="3-Jacksn"/>
<route tag="4" title="4 - Sutter" shortTitle="4-Sutter"/>
<route tag="5" title="5 - Fulton" shortTitle="5-Fulton"/>
<route tag="6" title="6 - Parnassus" shortTitle="6-Parnas"/>
<route tag="7" title="7 - Haight" shortTitle="7-Haight"/>
<route tag="14" title="14 - Mission" shortTitle="14-Missn"/>
<route tag="21" title="21 - Hayes" shortTitle="21-Hayes"/>
</body>
```

We can incorporate this API in our code like this:

```java
  private static Routes getRoutes(Agency ag) {
    try {
      final URL url = new URL(String.format(
          "https://retro.umoiq.com/service/publicXMLFeed?command=routeList&a=%s", ag.getId()));
      final Value allRoutes = parse(url);
      if (!allRoutes.isDefined()) {
        return null;
      }
      final Iterator<Item> it = allRoutes.iterator();
      final Routes routes = new Routes();
      while (it.hasNext()) {
        final Item item = it.next();
        final Value header = item.getAttr("route");
        if (header.isDefined()) {
          final Route route = new Route().withTag(header.get("tag").stringValue()).withTitle(header.get("title").stringValue());
          routes.add(route);
        }
      }
      return routes;
    } catch (Exception e) {
      log.severe(() -> String.format("Exception thrown:\n%s", e));
    }
    return null;
  }


  private static Value parse(URL url) {
    final HttpURLConnection urlConnection;
    try {
      urlConnection = (HttpURLConnection) url.openConnection();
      urlConnection.setRequestProperty("Accept-Encoding", "gzip, deflate");
      final InputStream stream = new GZIPInputStream(urlConnection.getInputStream());
      final Value configValue = Utf8.read(stream, Xml.structureParser().documentParser());
      return configValue;
    } catch (Throwable e) {
      log.severe(() -> String.format("Exception thrown:\n%s", e));
    }
    return Value.absent();
  }

```

###### <a name="vehicle-locations"></a>vehicleLocations

The second end-point corresponds to the `vehicleLocations` command, and will return a vehicle  object with fields for `id`, `routeTag`, `dirTag`, `lat`, `long`, `secsSinceReport`, `predictable` and `heading`. Here is an example request and response.

INPUT:

```
https://retro.umoiq.com/service/publicXMLFeed?command=vehicleLocations&a=sf-muni&r=N&t=1144953500233
```

OUTPUT:

```xml
<body>
<vehicle id="1453" routeTag="N" dirTag="out" lat="37.7664199" lon="-
122.44896" secsSinceReport="29" predictable="true" heading="276"/>
<vehicle id="1549" routeTag="N" dirTag="in" lat="37.77631" lon="-
122.3941" secsSinceReport="3" predictable="true" heading="45"/>
<vehicle id="1517" routeTag="N" dirTag="in_short" lat="37.76035" lon="-
122.50794" secsSinceReport="69" predictable="true" heading="267"/>
<vehicle id="1547" routeTag="N" dirTag="out" lat="37.76952" lon="-
122.43174" secsSinceReport="28" predictable="true" heading="85"/>
<vehicle id="1404" routeTag="N" dirTag="out" lat="37.76003" lon="-
122.50919" secsSinceReport="9" predictable="true" heading="117"/>
<vehicle id="1400" routeTag="N" dirTag="in" lat="37.76415" lon="-
122.46409" secsSinceReport="50" predictable="true" heading="266"/>
<lastTime time="1144953510433"/>
</body>
```

We can incorporate this API in our code like this:

```java

  public static Vehicles getVehicleLocations(String pollUrl, Agency ag) {
    try {
      final URL url = new URL(pollUrl);
      final Value vehicleLocs = parse(url);
      if (!vehicleLocs.isDefined()) {
        return null;
      }

      final Iterator<Item> it = vehicleLocs.iterator();
      final Vehicles vehicles = new Vehicles();
      while (it.hasNext()) {
        final Item item = it.next();
        final Value header = item.getAttr("vehicle");
        if (header.isDefined()) {
          final String id = header.get("id").stringValue().trim();
          final String routeTag = header.get("routeTag").stringValue();
          final float latitude = header.get("lat").floatValue(0.0f);
          final float longitude = header.get("lon").floatValue(0.0f);
          final int speed = header.get("speedKmHr").intValue(0);
          final int secsSinceReport = header.get("secsSinceReport").intValue(0);
          final String dir = header.get("dirTag").stringValue("");
          final String dirId;
          if (!dir.equals("")) {
            dirId = dir.contains("_0") ? "outbound" : "inbound";
          } else {
            dirId = "outbound";
          }

          final int headingInt = header.get("heading").intValue(0);
          String heading = "";
          if (headingInt < 23 || headingInt >= 338) {
            heading = "E";
          } else if (23 <= headingInt && headingInt < 68) {
            heading = "NE";
          } else if (68 <= headingInt && headingInt < 113) {
            heading = "N";
          } else if (113 <= headingInt && headingInt < 158) {
            heading = "NW";
          } else if (158 <= headingInt && headingInt < 203) {
            heading = "W";
          } else if (203 <= headingInt && headingInt < 248) {
            heading = "SW";
          } else if (248 <= headingInt && headingInt < 293) {
            heading = "S";
          } else if (293 <= headingInt && headingInt < 338) {
            heading = "SE";
          }
          final String uri = "/vehicle/" + ag.getCountry() + "/" + ag.getState() + "/" + ag.getId() + "/" + parseUri(id);
          final Vehicle vehicle = new Vehicle().withId(id).withUri(uri).withDirId(dirId).withIndex(ag.getIndex())
              .withLatitude(latitude).withLongitude(longitude).withRouteTag(routeTag).withSecsSinceReport(secsSinceReport)
              .withSpeed(speed).withHeading(heading);
          vehicles.add(vehicle);
        }
      }
      return vehicles;
    } catch (Exception e) {
      log.severe(() -> String.format("Exception thrown:\n%s", e));
    }
    return null;
  }

  private static String parseUri(String uri) {
    try {
      return java.net.URLEncoder.encode(uri, "UTF-8").toString();
    } catch (UnsupportedEncodingException e) {
      return null;
    }
  }
```

##### <a name="timers-tasks-and-polling"></a>Timers, tasks, and polling

### <a name="running-the-complete-demo"></a>Running the complete demo

#### Windows

Install the <a href="https://docs.microsoft.com/en-us/windows/wsl/install-win10">Windows Subsystem for Linux</a>.

Execute the command `./run.sh` from a console pointed to the application's home directory. This will start a Swim server, seeded with the application's logic, on port 9002.

   ```console
    user@machine:~$ ./run.sh
   ```

#### \*nix

Execute the command `./run.sh` from a console pointed to the application's home directory. This will start a Swim server, seeded with the application's logic, on port 9002.

   ```console
    user@machine:~$ ./run.sh
   ```

#### View the UI
Open the following URL on your browser: http://localhost:9002.

### <a name="running-the-complete-demo-as-fabric"></a>Running the complete demo as a fabric

Run two Swim instances on your local machine to distribute the applications Web Agents between the two processes.

```console
# Build the UI
server $ ./build.sh

# Start the first fabric node in one terminal window:
server $ ./gradlew run -Dswim.config.resource=server-a.recon

# Start the second fabric node in another terminal window:
server $ ./gradlew run -Dswim.config.resource=server-b.recon
```

When both processes are up and running, you can point your browser at either http://localhost:9008 (Server A) or http://localhost:9009 (Server B). You will see a live view of all Web Agents, regardless of which server you point your browser at. Swim transparently demultiplexes links opened by external clients, and routes them to the appropriate server in the fabric.
