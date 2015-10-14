# ``package.??``

Overview of package contents and functionality...

```java
public abstract Builder<T> {
	private List<BuildError<? extends T>> buildErrors = new ArrayList<>();
    private Builder.Callback<? extends T> builderCallback;
    
    public abstract Builder<T> build();
    public abstract <U extends T> Builder<T> build(Builder.Callback<U> builderCallback);
    
    public final void setCallback(Builder.Callback<? extends T> builderCallback) {
    	this.builderCallback = builderCallback;
    }
    
    protected final void addError(BuildError<? extends T> buildError) {
    	buildErrors.add(buildError);
    }
    
    protected final void beforeCommit() {
    	if (buildErrors.size() > 0) {
        	if (builderCallback != null) { // a callback was given, provide build errors
            	builderCallback.onBuildError(buildErrors);
            } else {
            	// no callback provided; throw the first error encountered
                buildErrors.get(0).throw();
            }
        } else {
        	// no build errors encountered; proceed with build
        }
    }
    
    public static interface Callback<T> {
    	void onBuildError(List<BuildError<? extends T>> buildErrors);
    }
}
```



# ``package.geofence``

Overview of package contents and functionality...

**Note:** See [GeofenceStatusCodes](https://developers.google.com/android/reference/com/google/android/gms/location/GeofenceStatusCodes). Package must be able to handle when

- The Geofence service is not available
- The app has created more than 100 geofences
- The app has provided more than 5 different PendingIntents to the [``addGeofences(GoogleApiClient, GeofencingRequest, PendingIntent)``](https://developers.google.com/android/reference/com/google/android/gms/location/GeofencingApi) call.

The first is beyond the control of the package. We can attempt to throttle the number of active Geofences by imposing (aggressive) limits on Geofence creation, and by issuing warnings to the application code when we are nearing the limit.

The last of these events should *never* occur if the package is written correctly, in that we can enforce the re-use of existing PendingIntents whenever an attempt to create a new one is made and the current number of different PendingIntents is at the limit.
To do so, all PendingIntents passed to ``addGeofences`` should be stored in a static container in the controller class for the package.

## ``GeofenceTools``

Description of this class...

#### ``String getStringId()``

#### ``int getIntId()``

#### ``UUID getUUID()``

### Implementation

```java
public final class GeofenceTools {
	public static UUID getUUID() {
    	return UUID.randomUUID();
    }
    
    public static String getStringId() {
    	return getUUID().toString();
    }
    
    public static int getIntId() {
    	return getUUID().hashCode();
    }
}
```

Some details about the implementation...



## ``AbstractGeofence``

Description of this class...

### Implementation

```java
public abstract class AbstractGeofence
	implements Geofence {

	private final String requestId;    
    private final double latitude;
    private final double longitude;
    private final float radius;
    private final int expirationMs;
    private final int loiteringDelay;
    private final int transitionFlags;
    private final int notificationResponsivenessMs;
    private final int initialTrigger;
    private final Class<? extends IntentService> intentServiceClass;
    private final Intent intent;
    
    public AbstractGeofence(
    		String requestId,
    		double latitude,
        	double longitude,
        	float radius,
        	int expirationMs,
        	int loiteringDelay,
        	int transitionFlags,
        	int notificationResponsivenessMs,
        	int initialTrigger,
        	Class<? extends IntentService> intentServiceClass,
        	Intent intent) {
    
    	// Resolve parameter constraints and null values

        if (requestId == null) {
        	this.requestId = GeofenceTools.getStringId();
        }
        
        ...
    	
	}
    
    protected class Builder {
    	private final String requestId;    
        private final double latitude;
        private final double longitude;
        private final float radius;
        private final int expirationMs;
        private final int loiteringDelay;
        private final int transitionFlags;
        private final int notificationResponsivenessMs;
        private final int initialTrigger;
        private final Class<? extends IntentService> intentServiceClass;
        private final Intent intent;
        
        private List<Exception> buildErrors = new ArrayList<>();
    
    	protected Builder() {}

        protected Builder(double latitude, double longitude, float radius) {
          this.latitude = latitude;
          this.longitude = longitude;
          this.radius = radius;
        }

        protected void error(Exception e) {
            buildErrors.add(e);
        }

        protected Builder setCircularRegion(double latitude, double longitude, float radius) {
            return this(latitude, longitude, radius);
        }

        protected Builder setCircularRegion(LatLng latLng, float radius) {
            return this(latLng.latitude, latLng.longitude, radius);
        }
    }
       
    @Override
    public String getRequestId() {
    	return requestId;
    }
}
```

Some details about the implementation...



## ``UpdatableGeofence``

### Notes

[4] Updating any properties of an ``UpdatableGeofence`` can be achieved in several ways:

- By manually creating a ``UpdatableGeofence.UpdateRequest`` using the associated ``UpdateRequestBuilder`` and passing it as the sole argument to the ``build`` method of the ``#.Builder``.
- By using the provided builder functions: ``setCircularRegion``, ``setTransitionTypes``, etc. (analogous to the functions provided by the default ``Geofence.Builder`` class; with some added functionality) as well as setting which properties are allowed to be updated at a later time, by calling ``updatable(int updatesAllowedFlag)``. ***Note**: The transition types specified using ``setTransitionTypes`` may be extended, depending on the allowable updates, to facilitate proper re-issuing of associated ``PendingIntent``s (see ``#.Radius``).*

[7]	An instance of the ``GoogleApiClient`` is generated when the library is instantiated, and is statically available to any class which is part of the library.

[8] When the instantiating an ``UpdatableGeofence`` using its ``Builder`` class, the events required to support the provided updates are automatically registrered with the ``EventController`` and the event handlers associated with them are attached to the object.
Internally this is achieved...

[9] Constraints on input parameters...

[9.1] Possible transition types...

### Implementation

```java
public class UpdatableGeofence
	extends AbstractGeofence
    implements Geofence {

	// The AbstractGeofence class holds logic for creation of appropriate GeofencingRequests and creation and sending of PendingIntents, as well as values necessary for creating a Geofence object. Also holds static logic for evaluating re-issuing for RADIUS and CENTER updatable types.
    
    private final byte updateFlags;
    
    public static final UPDATE_FILTER NONE			  = UPDATE_FILTER.NONE;
    public static final UPDATE_FILTER RADIUS		  = UPDATE_FILTER.RADIUS;
    public static final UPDATE_FILTER CENTER		  = UPDATE_FILTER.CENTER;
    public static final UPDATE_FILTER EXPIRATION	  = UPDATE_FILTER.EXPIRATION;
    public static final UPDATE_FILTER LOITERING_DELAY = UPDATE_FILTER.LOITERING_DELAY;
    public static final UPDATE_FILTER ALL			  = UPDATE_FILTER.ALL;
    
    private static enum UPDATE_FILTER {
    	NONE, RADIUS, CENTER, EXPIRATION, LOITERING_DELAY, ALL;
        
        public static final byte value;
        
        UPDATE() {
        	// Calculates the bit filter of the enum based on its order in the declaration
        	this.value = Math.pow(2, this.ordinal());
        }
    }
    
    private UpdatableGeofence(
    		String requestId,
        	byte updateFlags,
    		double latitude,
        	double longitude,
        	float radius,
        	int expirationMs,
        	int loiteringDelay,
        	int transitionFlags,
        	int notificationResponsivenessMs,
        	int initialTrigger,
        	Class<? extends IntentService> intentServiceClass,
        	Intent intent) {   

		super(requestId, latitude, longitude, radius, expirationMs, loiteringDelay, transitionFlags,
        		notificationResponsivenessMs, initialTrigger, intentServiceClass, intent);

        if (updateFlags == null) {
        	this.updateFlags = NONE;
        }
    }
    
    public class Builder extends AbstractGeofence.Builder {
    	private final String requestId;
        private final byte updateFlags;
    
    	private final double latitude;
        private final double longitude;
        private final float radius;
        private final int expirationMs;
        private final int loiteringDelay;
        private final int transitionFlags;
        private final int notificationResponsivenessMs;
        private final int initialTrigger;
        private final Class<? extends IntentService> intentServiceClass;
        private final Intent intent;
        
        
        
        public Builder setAvailableUpdates(UPDATE_FILTER... availableUpdates) {
        	byte flags = 0;
            
            for (i = 0; i < availableUpdates.length; i++) {
            	flags |= availableUpdates[i];
            }
            
            this.updateFlags = flags;
            
            return this;
        }
        
        ...
        
        public UpdatableGeofence build() {
        	// Enforce constraints on various attributes (e.g. expand transitionFlags given the updateFlags)
            
            int tmpTransitionFlags = transitionFlags;
            // If available updates include RADIUS or CENTER and transitions include DWELL, we must always also
            // include ENTER and EXIT transitions for the re-issuing logic to function correctly.
            if ((transitionFlags & Geofence.GEOFENCE_TRANSITION_DWELL)
            	&& (updateFlags & RADIUS == RADIUS)
            	|| (updateFlags & CENTER == CENTER)) {
            	
                tmpTransitionFlags |= Geofence.GEOFENCE_TRANSITION_ENTER | Geofence.GEOFENCE_TRANSITION_EXIT;
            }
            
            // The intent and intentServiceClass are mutually exclusive settings (where one or the other is
            // always requred; if this is not the case at this point, either add a BuildError<UpdatatableGeofence>
            // or throw it is an exception, if no callback was provided as a parameter.
            if (...)
        
        	return new UpdatableGeofence(
            	requestId, updateFlags, latitude, longitude, radius, expirationMs, loiteringDelay,
                tmpTransitionFlags, notificationResponsivenessMs, initialTrigger, intentServiceClass, intent);
        }
    }
}
```


### Examples
The following is a list of code examples illustrating the various features of the ``UpdatableGeofence`` class.

#### Creating an ``UpdatableGeofence``
The following code creates an ``UpdatableGeofence`` object using the provided builder functions.
The example assumes that ``expirationDate`` is a ``Date`` object, specifying the desired expiration time of the geofence.

```java
fence = new UpdatableGeofence.Builder()
				.setAvailableUpdates(RADIUS, CENTER, EXPIRATION)
                .setCircularRegion(
                	latitude,
                    longitude,
                    radius)
                .setExpiration(expirationDate)
                .build();
```
Note that in this example we do not use the ``setTransitionTypes`` function, letting the transitions that should cause event-triggers be determined from the context of the specified properties we wish it possible to update at a later stage.

