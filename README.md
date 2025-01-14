[![Flutter Community: firestore_helpers](https://fluttercommunity.dev/_github/header/firestore_helpers)](https://github.com/fluttercommunity/community)


>DISCLAIMER:
As some users pointed out, FireStore does not correctly compare GeoPoints at this time which lead to more data might be returned from the server than the server-side query intended. As long as you always also use the client-side filtering too by providing a distanceAccessor you will get the right results but if you get a lot of data entries the performance will degrade. Google wants to change that in the Future but no ETA yet. [Please see](https://github.com/firebase/firebase-js-sdk/issues/826#issuecomment-389598387)

> Warning: V3.0.0 has a breaking change as it fixes a typo clientSite <-> clientSide


# FirestoreHelpers

FireStore is a great database that is easy to work with. To make life even easier here is this package. It contains functions build queries dynamically and for location based queries.

The necessary math for the geographical calculations were ported from [this JS source on SO](https://stackoverflow.com/questions/46630507/how-to-run-a-geo-nearby-query-with-firestore) by [Stanton Parham](https://github.com/stparham)

## Creating Queries dynamically

In case you want to modify your queries at runtime `builtQuery()` might be helpful:

```Dart
/// 
/// Builds a query dynamically based on a list of [QueryConstraint] and orders the result based on a list of [OrderConstraint].
/// [collection] : the source collection for the new query
/// [constraints] : a list of constraints that should be applied to the [collection]. 
/// [orderBy] : a list of order constraints that should be applied to the [collection] after the filtering by [constraints] was done.
/// Important all limitation of FireStore apply for this method two on how you can query fields in collections and order them.
Query buildQuery({Query collection, List<QueryConstraint> constraints,
    List<OrderConstraint> orderBy})


/// Used by [buildQuery] to define a list of constraints. Important besides the [field] property not more than one of the others can ne [!=null].
/// They corespond to the possisble parameters of Firestore`s [where()] method. 
class QueryConstraint {
  QueryConstraint(
      {this.field,
      this.isEqualTo,
      this.isLessThan,
      this.isLessThanOrEqualTo,
      this.isGreaterThan,
      this.isGreaterThanOrEqualTo,
      this.isNull,
      this.arrayContains,
      this.arrayContainsAny,
      this.whereIn,
      this.whereNotIn});
}

/// Used by [buildQuery] to define how the results should be ordered. The fields 
/// corespond to the possisble parameters of Firestore`s [oderby()] method. 
class OrderConstraint {
  OrderConstraint(this.field, this.descending);
}
```


### Example

Lets assume we have an events collection in FireStore and we want to get all events that apply to certain constraints:


```Dart

// Let's assume that our Event class looks like this 

class Event{
  String id;
  String name;
  DateTime startTime;
  GeoPoint location;
}


Stream<List<Event>> getEvents({List<QueryConstraint> constraints}) {
  try {
    Query ref = buildQuery(
      collection: eventCollection, 
      constraints: constraints, orderBy: [
          new OrderConstraint("startTime", false),
        ]);
    return ref.snapshots().map((snapShot) => snapShot.documents.map(eventDoc) {
          var event = _eventSerializer.fromMap(eventDoc.data);
          event.id = eventDoc.documentID;
          return event;
        }).toList());
  } on Exception catch (ex) {
    print(ex);
  }
  return null;
}

/// And gets called somewhere else

 getEvents(constraints: [new QueryConstraint(field: "creatorId", isEqualTo: _currentUser.id)]);

```


To make this even more comfortable and powerful there is `getDataFromQuery()`

```Dart
typedef DocumentMapper<T> = T Function(DocumentSnapshot document);
typedef ItemFilter<T> = bool Function(T);
typedef ItemComparer<T> = int Function(T item1, T item2);

///
/// Convenience Method to access the data of a Query as a stream while applying 
/// a mapping function on each document with optional client side filtering and sorting
/// [qery] : the data source
/// [mapper] : mapping function that gets applied to every document in the query. 
/// Typically used to deserialize the Map returned from FireStore
/// [clientSideFilters] : optional list of filter functions that execute a `.where()` 
/// on the result on the client side
/// [orderComparer] : optional comparisson function. If provided your resulting data 
/// will be sorted based on it on the client

Stream<List<T>> getDataFromQuery<T>({
  Query query,
  DocumentMapper<T> mapper,
  List<ItemFilter> clientSidefilters,
  ItemComparer<T> orderComparer,
});
```

With this our example get this with some additional functionality:

We want the events to be ordered by name and only future events. 
In this example we don't do the sorting on the server but on client side after the filtering

```Dart
Stream<List<Event>> getEvents({List<QueryConstraint> constraints}) {
  try {
    Query query = buildQuery(collection: eventCollection, constraints: constraints);
    return getDataFromQuery(
        query: query, 
        mapper: (eventDoc) {
          var event = _eventSerializer.fromMap(eventDoc.data);
          event.id = eventDoc.documentID;
          return event;
        }, 
        clientSidefilters: (event) => event.startTime > DateTime.now()  // only future events
        orderComparer: (event1, event2) => event1.name.compareTo(event2.name) 
      );
  } on Exception catch (ex) {
    print(ex);
  }
  return null;
}
```

## Location based queries

>DISCLAIMER:
As some users pointed out, FireStore does not correctly compare GeoPoints at this time which lead to more data might be returned from the server than the server-side query intended. As long as you always also use the client-side filtering too by providing a distanceAccessor you will get the right results but if you get a lot of data entries the performance will degrade. Google wants to change that in the Future but no ETA yet. [Please see](https://github.com/firebase/firebase-js-sdk/issues/826#issuecomment-389598387)

A quite common scenario in an mobile App is to query for data that's location entry matches a certain search area.
Unfortunately FireStore doesn't support real geographical queries, but we can query _less than_ and _greater than_ on `GeopPoints`. Which allows to span a search square defined by its south-west and north-east corners.

As most App require to define a search area by a centre point  and a radius we have `calculateBoundingBoxCoordinates`

```Dart
/// Defines the boundingbox for the query based
/// on its south-west and north-east corners
class GeoBoundingBox {
  final GeoPoint swCorner;
  final GeoPoint neCorner;

  GeoBoundingBox({this.swCorner, this.neCorner});
}

///
/// Defines the search area by a  circle [center] / [radiusInKilometers]
/// Based on the limitations of FireStore we can only search in rectangles
/// which means that from this definition a final search square is calculated
/// that contains the circle
class Area {
  final GeoPoint center;
  final double radiusInKilometers;

  Area(this.center, this.radiusInKilometers): 
  assert(geoPointValid(center)), assert(radiusInKilometers >= 0);

  factory Area.inMeters(GeoPoint gp, int radiusInMeters) {
    return new Area(gp, radiusInMeters / 1000.0);
  }

  factory Area.inMiles(GeoPoint gp, int radiusMiles) {
    return new Area(gp, radiusMiles * 1.60934);
  }

  /// returns the distance in km of [point] to center
  double distanceToCenter(GeoPoint point) {
    return distanceInKilometers(center, point);
  }
}

///
///Calculates the SW and NE corners of a bounding box around a center point for a given radius;
/// [area] with the center given as .latitude and .longitude
/// and the radius of the box (in kilometers)
GeoBoundingBox boundingBoxCoordinates(Area area)
```

If you use `buildQuery()` is even gets easier with `getLocationsConstraint`


```Dart
/// Creates the necessary constraints to query for items in a FireStore collection that are inside a specific range from a center point
/// [fieldName] : the name of the field in FireStore where the location of the items is stored
/// [area] : Area within that the returned items should be
List<QueryConstraint> getLocationsConstraint(String fieldName, Area area) 
```


```Dart
/// function type used to acces the field that contains the loaction inside 
/// the generic type
typedef LocationAccessor<T> = GeoPoint Function(T item);

/// function typse used to access the distance field that contains the 
/// distance to the target inside the generic type
typedef DistanceAccessor<T> = double Function(T item);

typedef DistanceMapper<T> = T Function(T item, double itemsDistance);
```

`getDataInArea()` combines all the above functions to one extremely powerful function:

```Dart
///
/// Provides as Stream of lists of data items of type [T] that have a location field in a
/// specified area sorted by the distance of to the areas center.
/// [area]  : The area that constraints the query
/// [source] : The source FireStore document collection
/// [mapper] : mapping function that gets applied to every document in the query.
/// Typically used to deserialize the Map returned from FireStore
/// [locationFieldInDb] : The name of the data field in your FireStore document.
/// Need to make the location based search on the server side
/// [locationAccessor] : As this is a generic function it cannot know where your
/// location is stored in you generic type.
/// optional if you don't use [distanceMapper] and don't want to sort by distance
/// Therefore pass a function that returns a valur from the location field inside
/// your generic type.
/// [distanceMapper] : optional mapper that gets the distance to the center of the
/// area passed to give you the chance to save this inside your item
/// if you use a [distanceMapper] you HAVE to pass [locationAccessor]
/// [clientSideFilters] : optional list of filter functions that execute a `.where()`
/// on the result on the client side
/// [distanceAccessor] : if you have stored the distance using a [distanceMapper] passing
/// this accessor function will prevent additional distance computing for sorting.
/// [sortDecending] : if the resulting list should be sorted descending by the distance
/// to the area's center. If you don't provide [loacationAccessor] or [distanceAccessor]
/// no sorting is done. This Sorting is done one the client side
/// [serverSideConstraints] : If you need some serverside filtering besides the [Area] pass a list of [QueryConstraint] 
/// [serverSideOrdering] : If you need some serverside ordering you can pass a List of [OrderConstraints]  
/// Using [serverSideConstraints] or  [serverSideOrdering] almost always requires to create an index for 
/// this field. Check your debug output for a message from FireStore with
/// a link to create them
Stream<List<T>> getDataInArea<T>(
    {@required Area area,
    @required Query source,
    @required DocumentMapper<T> mapper,
    @required String locationFieldNameInDB,
    LocationAccessor<T> locationAccessor,
    List<ItemFilter<T>> clientSidefilters,
    DistanceMapper<T> distanceMapper,
    DistanceAccessor<T> distanceAccessor,
    bool sortDecending = false,
    List<QueryConstraint> serverSideConstraints,
    List<OrderConstraint> serverSideOrdering})
```

Best to see an example how we would use it:


```Dart
// We will define a wrapper class because we want our Event plut its distance back

class EventData{
  Event event;
  double distance;

  EventData(this.event, [this.distance]) 
}



Stream<List<EventData>> getEvents(area) {
  try {
    return getDataInArea(
        collection: Firestore.instance.collection("events"),
        area: area,
        locationFieldNameInDB: 'location',        
        mapper: (eventDoc) {
          var event = _eventSerializer.fromMap(eventDoc.data);
          // if you serializer does not pass types like GeoPoint through
          // you have to add that fields manually. If using `jaguar_serializer` 
          // add @pass attribute to the GeoPoint field and you can omit this. 
          event.location = eventDoc.data['location'] as GeoPoint;
          event.id = eventDoc.documentID;
          return new EventData(event);
        },
        locationAccessor: (eventData) => eventData.event.location,
        distanceMapper: (eventData, distance) {
          eventData.distance = distance;
          return eventData;
        },
        distanceAccessor: (eventData) => eventDatas.distance, 
        clientSidefilters: (event) => event.startTime > DateTime.now()  // filer only future events
      );
  } on Exception catch (ex) {
    print(ex);
  }
  return null;
}
```


**IMPORTANT** to enable FireStore to execute queries based on `GeopPoints` you can not serialize the GeoPoints before you hand them to FireStore's `setData` if you use a code generator that does not allow to mark certain field as passthrough you have to set the value manually like here. If using `jaguar_serializer`  add `@pass` attribute to the GeoPoint field and you can omit this.

```Dart
  Future<bool> updateEvent(Event event) async {
    try {
      var eventData = _eventSerializer.toMap(event);
  ->  eventData['location'] = event.location;
      await eventCollection.document(event.id).setData(eventData);
      return true;
    } catch (e, stack) {
      print(e);
      print(stack.toString());
      //todo logging
      return false;
    }
  }
```

I use [jaguar_serializer](https://pub.dartlang.org/flutter/packages?q=jaguar_serializer+) which is great in combination with FireStore because it produces a `Map<String, dynamic>` instead of JSON string. To make not to encode GeoPoints but pass them through just add `@pass` attribute to your `GeoPoint` fields.
