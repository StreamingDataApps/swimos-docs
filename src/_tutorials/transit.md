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

We’ll be using Gradle to build the application, installation instructions can be found <a href="https://gradle.org/install/">here</a>. We'll start with some boilerplate that will generally require little changes across projects. First, we will download the gradle files from this <a href="https://github.com/swimos/project-starter/blob/main/java/gradle-files.tgz">tarball</a> or <a href="https://github.com/swimos/project-starter/blob/main/java/gradle-files.zip">zip archive</a>.

Then extract as appropriate to the server directory:
- `tar zxvf ~/Downloads/gradle-files.tgz`
- `unzip ~/Downloads/gradle-files.zip`

copy `build.gradle` to our `server` directory. You can find a copy here, with the section most likely to change highlighted:

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
  - gradle.properties
  - settings.gradle

To fill out the Java application structure under `server`, just do the following:

```console
mkdir -p src/main/java/swim/transit/agent
mkdir -p src/main/resources
```

#### <a name="creating-initial-files"></a>Creating the app configuration files

We will be creating the following configuration file: `server/src/main/resources/server.recon`

```java
@kernel(class: 'swim.reflect.ReflectKernel', optional: true)

transit: @fabric {
    @node {
        pattern: "/country/:id"
        @agent(class: "swim.transit.agent.CountryAgent")
    }
    @node {
        pattern: "/state/:country/:state"
        @agent(class: "swim.transit.agent.StateAgent")
    }
    @node {
        pattern: "/agency/:country/:state/:id"
        @agent(class: "swim.transit.agent.AgencyAgent")
    }
    @node {
        pattern: "/vehicle/:country/:state/:agency/:id"
        @agent(class: "swim.transit.agent.VehicleAgent")
    }
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

We will be using a CSV file to load transit information for 46 transit agencies into Web Agents:
- https://raw.githubusercontent.com/swimos/transit/master/server/src/main/resources/agencies.csv

This CSV data file contains the fields for each agency: `id`, `state`, and `country`. It should be saved under `server/src/main/resources/`.

### <a name="first-contact"></a>First Contact

We will implement Web Agents for each of our entities under `server/src/main/java/swim/transit/agent/`:
- `AgencyAgent.java`
- `CountryAgent.java`
- `StateAgent.java`
- `VehicleAgent.java`

#### <a name="creating-a-web-agent"></a>Creating a Web Agent

We will now create a barebones Web Agent for the vehicle entity. To do that, we’ll start by extending `AbstractAgent` and override the `didStart()` method:

```java
package swim.transit.agent;

import swim.api.agent.AbstractAgent;
import java.util.logging.Logger;

public class VehicleAgent extends AbstractAgent {
  private static final Logger log = Logger.getLogger(VehicleAgent.class.getName());

  @Override
  public void didStart() {
    log.info(()-> String.format("Started Agent: %s", nodeUri()));
  }

}
```

#### <a name="creating-the-app-plane"></a>Creating the App Plane

We will define our application entry point under `server/src/main/java/swim/transit/` as `TransitPlane.java`. We can do this by extending `TransitPlane` from the `AbstractPlane` base class, which will allow us to declare the application routes. The agent corresponding to each route is declared using the `@SwimAgent` annotation. The application routes are declared using the `@SwimRoute` annotation. The route itself is defined via the generic `AgentRoute` type.

```java
package swim.transit;

import swim.api.*;
import swim.api.plane.AbstractPlane;
import swim.api.space.Space;
import swim.kernel.Kernel;
import swim.server.ServerLoader;
import swim.transit.agent.VehicleAgent;
import swim.structure.Value;
import java.util.logging.Logger;

public class TransitPlane extends AbstractPlane {
  private static final Logger log = Logger.getLogger(TransitPlane.class.getName());

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
import swim.api.SwimLane;
import swim.api.lane.CommandLane;
import swim.api.lane.ValueLane;
import swim.recon.Recon;
import swim.structure.Record;
import swim.structure.Value;

...

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

We'll now modify `VehicleAgent` a bit more to derive acceleration from time and speed, as well as store the last 10 speed and acceleration data points.

```java
  private long lastReportedTime = 0L;

  @SwimLane("speeds")
  public MapLane<Long, Integer> speeds;

  @SwimLane("accelerations")
  public MapLane<Long, Integer> accelerations;
```

You can now modify `onUpdateVehicle` to handle acceleration and speed. 

```java
import swim.api.lane.MapLane;

...

 private void onUpdateVehicle(Value v) {
    final Value oldState = vehicle.get();
    final long time = System.currentTimeMillis() - (v.get("secsSinceReport").longValue(0) * 1000L);
    final int oldSpeed = oldState.isDefined() ? oldState.get("speed").intValue(0) : 0;
    final int newSpeed = v.get("speed").intValue(0);

    this.vehicle.set(v);
    speeds.put(time, newSpeed);
    if (speeds.size() > 10) {
      speeds.drop(speeds.size() - 10);
    }
    if (lastReportedTime > 0) {
      final float acceleration = (float) ((newSpeed - oldSpeed)) / (time - lastReportedTime) * 3600;
      accelerations.put(time, Math.round(acceleration));
      if (accelerations.size() > 10) {
        accelerations.drop(accelerations.size() - 10);
      }
    }
    lastReportedTime = time;
  }
```

Let’s re-run our code to observe recordings of speed and derivations of acceleration:
```console
./gradlew build
./gradlew run
```

### <a name="implement-agents-for-transit-domain"></a>Implement agents for transit domain

We’ve already implemented VehicleAgent, so we’ll move on to the remaining agents for state, country, and agent.

#### <a name="implement-agency-agent"></a>Implement AgencyAgent

##### <a name="Implement NextBusHttpAPI transit wrapper"></a>Implement NextBusHttpAPI transit wrapper

There are two public transit APIs we will connect to from UmoIQ (https://retro.umoiq.com/xmlFeedDocs/NextBusXMLFeed.pdf):
https://retro.umoiq.com/service/publicXMLFeed?command=routeList&a={agencyId}
https://retro.umoiq.com/service/publicXMLFeed?command=vehicleLocations&a={agencyId}&t=0

We will encapsulate this functionality with a wrapper, `NextBusHttpAPI.java`, that will sit alongside TransitPlane.java under `server/src/main/java/swim/transit/NextBusHttpAPI.java`. The first end-point corresponds to the `routeList` command, and will return a route object with a `tag`, `title`, and `shortTitle`. We will make use of the tag and title fields. 

```java
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Iterator;
import java.util.logging.Logger;
import java.util.zip.GZIPInputStream;
import swim.codec.Utf8;
import swim.structure.Item;
import swim.structure.Record;
import swim.structure.Value;
import swim.xml.Xml;

public class NextBusHttpAPI {
    private static final Logger log = Logger.getLogger(NextBusHttpAPI.class.getName());
    private NextBusHttpAPI() { }


  private static Value getRoutes(Value ag) {
      try {
          final URL url = new URL(String.format(
                  "https://retro.umoiq.com/service/publicXMLFeed?command=routeList&a=%s", ag.get("id").stringValue()));
          final Value allRoutes = parse(url);
          if (!allRoutes.isDefined()) {
              return null;
          }
          final Iterator<Item> it = allRoutes.iterator();
          final Record routes = Record.of();
          while (it.hasNext()) {
              final Item item = it.next();
              final Value header = item.getAttr("route");
              if (header.isDefined()) {
                  final Value route = Record.of()
                          .slot("tag", header.get("tag").stringValue())
                          .slot("title", header.get("title").stringValue());
                  routes.item(route);
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
    
}
```

The second end-point corresponds to the `vehicleLocations` command, and will return a vehicle  object with fields for `id`, `routeTag`, `dirTag`, `lat`, `long`, `secsSinceReport`, `predictable` and `heading`. 

```java
  public static Value getVehicleLocations(String pollUrl, Value ag) {
  try {
      final URL url = new URL(pollUrl);
      final Value vehicleLocs = parse(url);
      if (!vehicleLocs.isDefined()) {
          return null;
      }

      final Iterator<Item> it = vehicleLocs.iterator();
      final Record vehicles = Record.of();
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
              final String uri = "/vehicle/" +
                      ag.get("country").stringValue() +
                      "/" + ag.get("state").stringValue() +
                      "/" + ag.get("id").stringValue() +
                      "/" + parseUri(id);
              final Record vehicle = Record.of()
                      .slot("id", id)
                      .slot("uri", uri)
                      .slot("dirId", dirId)
                      .slot("index", ag.get("index").intValue())
                      .slot("latitude", latitude)
                      .slot("longitude", longitude)
                      .slot("routeTag", routeTag)
                      .slot("secsSinceReport", secsSinceReport)
                      .slot("speed", speed)
                      .slot("heading", heading);
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

#### <a name="implement-state-agent"></a>Implement StateAgent

Let’s implement StateAgent to manage the agencies and vehicles operating within the State. The basic lanes we’ll create maintain high-level statistics and element collections.

```java
package swim.transit.agent;

import java.util.Iterator;
import java.util.logging.Logger;

import swim.api.SwimLane;
import swim.api.agent.AbstractAgent;
import swim.api.lane.CommandLane;
import swim.api.lane.JoinMapLane;
import swim.api.lane.JoinValueLane;
import swim.api.lane.MapLane;
import swim.api.lane.ValueLane;
import swim.structure.Record;
import swim.structure.Value;

import java.util.logging.Logger;

public class StateAgent extends AbstractAgent {
  private static final Logger log = Logger.getLogger(StateAgent.class.getName());
    
  @SwimLane("count")
  public ValueLane<Value> count;

  @SwimLane("agencyCount")
  public MapLane<Value, Integer> agencyCount;

  @SwimLane("vehicles")
  public MapLane<String, Value> vehicles;

 @SwimLane("speed")
  public ValueLane<Float> speed;

  @SwimLane("agencySpeed")
  public MapLane<Value, Float> agencySpeed;
}
```

The join lanes allow us to link to other agents, such as Agency and Vehicle. We’ll make use of SwimOS’s `JoinValueLane` to reflect the vehicle count and average vehicle speed for each agency. We’ll reflect the topology of agencies and vehicles by aggregating each agency and each agency’s vehicles. Then, we’ll link to an agency with respect to specific lanes with `addAgency()`, while making use of AbstractAgent’s context object to send commands to other Web Agents. We will materialize the links using the downlink command exposed on all join lane types, passing in an Agency data object as the key:

```java
joinAgencySpeed.downlink(agency).nodeUri(agency.getUri()).laneUri("speed").open();
```

Here is the relevant code for the StateAgent:

```java
  @SwimLane("joinAgencyCount")
  public JoinValueLane<Value, Integer> joinAgencyCount = this.<Value, Integer>joinValueLane()
          .didUpdate(this::updateCounts);

  public void updateCounts(Value agency, int newCount, int oldCount) {
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
  public JoinValueLane<Value, Float> joinAgencySpeed = this.<Value, Float>joinValueLane()
          .didUpdate(this::updateSpeeds);

  public void updateSpeeds(Value agency, float newSpeed, float oldSpeed) {
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
  public JoinMapLane<Value, String, Value> joinAgencyVehicles = this.<Value, String, Value>joinMapLane()
          .didUpdate((String key, Value newEntry, Value oldEntry) -> vehicles.put(key, newEntry))
          .didRemove((String key, Value vehicle) -> vehicles.remove(key));

  @SwimLane("addAgency")
  public CommandLane<Value> agencyAdd = this.<Value>commandLane().onCommand((Value agency) -> {
      log.info("uri: " + agency.get("uri").stringValue());
      joinAgencyCount.downlink(agency).nodeUri(agency.get("uri").stringValue()).laneUri("count").open();
      joinAgencyVehicles.downlink(agency).nodeUri(agency.get("uri").stringValue()).laneUri("vehicles").open();
      joinAgencySpeed.downlink(agency).nodeUri(agency.get("uri").stringValue()).laneUri("speed").open();
      // String id, String state, String country, int index
      Record newAgency = Record.of()
              .slot("id", agency.get("id").stringValue())
              .slot("state", agency.get("state").stringValue())
              .slot("country", agency.get("country").stringValue())
              .slot("index", agency.get("index").intValue())
              .slot("stateUri", nodeUri().toString());
      context.command("/country/" + getProp("country").stringValue(), "addAgency", newAgency);
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
