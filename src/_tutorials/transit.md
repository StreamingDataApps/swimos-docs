---
title: Transit Tutorial
layout: page
---

The Transit Tutorial walks you step-by-step through creating a small, but fully functional backend application for conveying real-time city transit information. The application involves four types of business entities: vehicles, agencies, states, and countries. These entities form a simple hierarchy where each vehicle falls under exactly one of 46 agencies. Each agency likewise falls under exactly one state, and each state falls under exactly one country.

Rather than simulating data, we will be utilizing an API provided by Cubic Transportation System's Umo Mobility Platform to retrieve real-time transit information -- NextBus, https://retro.umoiq.com/.

### <a name="project-creation"></a>Project Creation

Let’s start by creating the root project folder. We are calling the directory `transit` and the `server` sub-directory.

```console
$ mkdir -p transit/server
$ cd transit/server
```

#### <a name="prerequisites"></a>Prerequisites

To build this application, we'll need the JDK for <a href="https://www.oracle.com/technetwork/java/javase/downloads/index.html">Java 11+</a>. Click <a href="https://www.oracle.com/technetwork/java/javase/downloads/index.html">here</a> for JDK installation instructions. In conjunction with this, make sure your `JAVA_HOME` environment variable is pointed to your Java installation location, and that your `PATH` includes `JAVA_HOME`.

#### <a name="gradle-setup"></a>Building with Gradle

We’ll be using Gradle to build the application. Click here for installation instructions. We'll start with some boilerplate that will generally require little changes across projects. First, we will copy `build.gradle` to our `server` directory. You can find a copy here, with the section most likely to change highlighted:

- https://github.com/swimos/transit/blob/e9ee859e9e768db47cf2b491b573aa3100642062/server/build.gradle#L22-L25

Next, we'll copy gradle.properties to the same location:
- https://github.com/swimos/transit/blob/e9ee859e9e768db47cf2b491b573aa3100642062/server/gradle.properties
- https://github.com/swimos/transit/blob/e9ee859e9e768db47cf2b491b573aa3100642062/server/gradlew
- https://github.com/swimos/transit/blob/e9ee859e9e768db47cf2b491b573aa3100642062/server/gradlew.bat
- https://github.com/swimos/transit/blob/e9ee859e9e768db47cf2b491b573aa3100642062/server/settings.gradle
- https://github.com/swimos/transit/tree/e9ee859e9e768db47cf2b491b573aa3100642062/server/gradle/wrapper

`settings.gradle` specifies the root project name. `gradlew` and `gradlew.bat` are wrappers that should not generally be modified. `gradle.properties` specifies the version of the application and the swim libraries. The `gradle` directory contains the `wrapper` sub-directory which contains dependencies for `gradlew` and `gradlew.bat`, which stream-line dependency issues when using gradle. 

### <a name="project-organization"></a>Project organization

#### <a name="directory-structure"></a>Directory structure

From the root project directory, the directory structure should currently look like this:

- server
  - build.gradle
  - gradle
    - wrapper
      - gradle-wrapper.jar
      - gradle-wrapper.properties
    - gradlew
    - gradlew.bat
    - settings.gradle

To fill out the Java application structure under `server`, just do the following:

```console
mkdir -p src/main/java/swim/transit/agent
mkdir -p src/main/resources
```

#### <a name="creating-initial-files"></a>Creating the app configuration files

We will be creating the following files:
- `server/src/main/resources/server.recon`
- `server/src/main/java/module-info.java`

We’ll create `server.recon` under the `resources` directory and add the following code:

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

`@kernel` is used to specify kernel properties that determine runtime settings. Then, within the `@fabric` definition, we specify the Java class that will serve as our application entry point, `TransitPlane`. We define @store to use the file system as a lightweight backing store, though in-memory remains the primary state store.

Lastly, we need to configure the client-facing end-point. We do that using the @web directive where we set the port to `9001`. We tie the end-point to the fabric using the `space` property so it names the fabric’s key. We turn off websocket compression by setting `serverCompressionLevel` and `clientCompressionLevel` to 0.

For `module-info.java` which specifies internal library dependencies, dependencies for our backend service, and the service implementation our service provides, we'll specify the following code:

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

We will be using a CSV file to load transit information for 46 transit agencies into Web Agents:
- https://raw.githubusercontent.com/swimos/transit/master/server/src/main/resources/agencies.csv

This CSV data file contains the fields for each agency: `id`, `state`, and `country`. It should be saved under `server/src/main/resources/`.

### <a name="first-contact"></a>First Contact

We will implement Web Agents for each of our entities under `server/src/main/java/swim/transit/agent/`:
- `Agency.java`
- `Country.java`
- `State.java`
- `Vehicle.java`

We will define our application entry point under `server/src/main/java/swim/transit/` as `TransitPlane.java`.

#### <a name="creating-the-app-plane"></a>Creating the App Plane

We begin by extending `TransitPlane` from the `AbstractPlane` base class, which will allow us to declare the application routes. The agent corresponding to each route is declared using the `@SwimAgent` annotation. The application routes are declared using the `@SwimRoute` annotation. The route itself is defined via the generic `AgentRoute` type.

```java
package swim.transit;

import swim.api.*;
import swim.kernel.Kernel;
import swim.server.ServerLoader;
import swim.transit.agent.VehicleAgent;
import swim.structure.Value;
import java.util.logging.Logger;

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
```

We'll see that our plane has started and that the VehicleAgent is running.

#### <a name="handling-a-command"></a>Handling a Command

We will define a CommandLane that receives commands to update the state managed by a VehicleAgent whenever new values for that state are returned by the traffic API. Our vehicle data has been stored in a `Value` object that exposes `get()` methods to retrieve data from a generic structure. We will store the `Value` by defining a `ValueLane` and adding a handler that logs each state change.

```java

  @SwimLane("vehicle")
  public ValueLane<Value> vehicle = this.<Value>valueLane()
          .didSet((newValue, oldValue) -> {
            log.info("vehicle changed from " + Recon.toString(newValue) + " from " + Recon.toString(oldValue));
          });

  @SwimLane("updateVehicle")
  public CommandLane<Value> addVehicle = this.<Value>commandLane().onCommand(this::onUpdateVehicle);

  private void onUpdateVehicle(Value v) {
    this.vehicle.set(v);
  }

```

Lastly, we’ll invoke `updateVehicle` from the TransitPlane changing the command lane from `fake` to `updateVehicle` so that the line below

```java
space.command("/vehicle/US/CA/poseurs/dummy", "fake", Value.empty());
```

becomes

```java
    Record dummyVehicleInfo = Record.of()
            .slot("id", "8888")
            .slot("uri", "/vehicle/US/CA/poseurs/dummy/8888")
            .slot("dirId", "outbound")
            .slot("index", 26)
            .slot("latitude", 34.07749)
            .slot("longitude", -117.44896)
            .slot("routeTag", "61")
            .slot("secsSinceReport", 9)
            .slot("speed", 0)
            .slot("heading", "N");

    space.command("/vehicle/US/CA/poseurs/dummy", "updateVehicle", dummyVehicleInfo);
```

Think of `Record` as a `JSON object`, with slots being the key-value pairs. Let’s re-run now to confirm that our command has been received and the state has been stored:
```console
./gradlew build
./gradlew run
```

#### <a name="acting-on-state-changes"></a>Acting on state changes

We will now great some lanes that hold typed state so that we can process the old and new state as new state changes stream in. In the previous iteration, we sent a command that included Vehicle state as input. Let’s store that state now for use within the agent as well as any external observers that need to follow these state changes. We’ll create a SwimLane named vehicle that will specify the externally accessible route along with the underlying state field of type `ValueLane<Vehicle>` so that we can retrieve state via get() and set(). Instead of simply overwriting the `speed` field for Vehicle state, we will store the last 10 speeds indexed by timestamp and derive acceleration using the SwimOS Map counterpart, MapLane, which provides `get()` and `put()` accessors. We’ll apply the @SwimTransient tag to the speeds and acceleration to opt out of writing to disk since there is no long-term or fault tolerance benefit to tracking current acceleration.

We will derive acceleration from time and speed, and maintain the last 10 speed and acceleration readings.

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

We will now complete the implementation of the domain models, starting with the three principle models `Agency`, `Route`, and `Vehicle`. Agency provides a logical view of a specific transit system.

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

We’ve already implemented VehicleAgent, so we’ll move on to the remaining agents for state, country, and agent.

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
