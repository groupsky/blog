---
title: Google Maps with bearing
date: 2016-04-20 10:10:59
categories:
- Android
tags:
- android
- google maps
- sensors
- compass
---

Recently I had to implement [Google Maps][google-maps] for an Android app that will display the device location and bearing. 
I was thinking it would be a stright forward experience, but as always on android I stumbled upon several obstacles.

## Current location
The current location is quite easy to display, it's just a matter of a simple call [`mMap.setMyLocationEnabled(true)`][set-my-location-enabled]. 

So far so good, the position is displayed along with the circle displaying the accuracy, but where is my bearing? 
I tried to find some additional configuration but to my surprise there was none. Further research showed that
Google Maps will display the bearing arrow but needs the bearing provided by the [`LocationSource`][location-source].

## Bearing
Alright, guess I'll just need to stitch one `LocationSource` myself. First thing is to get the location updates, 
so I opened the [tutorial for location][location-tutorial]. There you are referenced to the tutorial for setting up 
google play services as well as the getting the last known location. 

### Obstacle 1
All of the tutorials failed to mention you also need to include a very important dependency `compile 'com.google.android.gms:play-services-location:8.3.0'`, 
which was manifested by not being able to resolve `LocationServices`.

### Obstacle 2
After following the tutorial to getting the location I managed to get a simple `LocationSource` that would emit location 
when one is received. But when tested the location didn't change. After some digging I found that looking only at the code
in the tutorial is lacking an important piece - initialization of the `LocationRequest` which by default triggers only
a single location update.

### Obstacle 3
Now I was at the same situation as before - I got the location displayed, but no bearing. So I googled around to see
how do you determine the bearing - turns out it used to be simple by using the `TYPE_ORIENTATION` but it got deprecated 
and instead you need to calculate that yourself using the `TYPE_ACCELEROMETER` and `TYPE_MAGNETIC_FIELD` sensors,
which boils down to waiting to get data from both sensors, and then perform some hard to understand transformations on it
just to get the azimuth (the angle between the magnetic north pole and the device up). Here's some code since most of the
sample in the net weren't production ready:

```java
    // These are better off as class fields to optimize garbage collection
    private final float[] mValuesAccelerometer;
    private final float[] mValuesMagneticField;
    private final float[] mMatrixR;
    private final float[] mMatrixI;
    private final float[] mMatrixValues;

    // ok this may easily be skraped, but for clarity I'm including it
    private float mAzimuthRadians = Float.NaN;
    // this one will hold our azimuth in degrees
    private float mAzimuth = Float.NaN;
    
    // these two are meant to skip useless calculations until we get data from both sensors
    private boolean mHasAccelerometerValues = false;
    private boolean mHasMagneticValues = false;

    @Override
    public void onSensorChanged(SensorEvent event) {
        switch (event.sensor.getType()) {
            case Sensor.TYPE_ACCELEROMETER:
                System.arraycopy(event.values, 0, mValuesAccelerometer, 0, 3);
                mHasAccelerometerValues = true;
                break;
            case Sensor.TYPE_MAGNETIC_FIELD:
                System.arraycopy(event.values, 0, mValuesMagneticField, 0, 3);
                mHasMagneticValues = true;
                break;
        }

        if (mHasAccelerometerValues && mHasMagneticValues) {
            if (SensorManager.getRotationMatrix(mMatrixR, mMatrixI, mValuesAccelerometer, mValuesMagneticField)) {
                SensorManager.getOrientation(mMatrixR, mMatrixValues);

                mAzimuthRadians = mMatrixValues[0];

                mAzimuth = (float) Math.toDegrees(mAzimuthRadians);

                // let anyone interested that we have a new azimuth
            }
        }
    }

```

### Obstacle 4
So after I got my azimuth I just needed to start producing `Location`s with bearing:

```java
    @Override
    public void onLocationChanged(Location location) {
        mLocation = location;

        notifyListener();
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        // .... skiped the code from previous sample
        
        notifyListener();
    }
    
    private void notifyListener() {
        if (mListener != null && mLocation != null) {
            // update the bearing if we have one
            if (mAzimuth != Float.NaN) {
                mLocation.setBearing(mAzimuth);
            }

            //The listener will be the maps:
            mListener.onLocationChanged(mLocation);
        }
    }
```

Now this looks quite ready, and in fact Google Maps did display my bearing, but when tested on device the bearing was
so chaotic and jumpy it did not seems usable. So I needed to make thinks more smooth. Going back to googling I found 
one particular [article][smooth-article] that was not only producing smooth values but also did the extra mile of calculating
the true north.

## Playing nice
Finally I had something that appears to be working, and now I need to take care of properly handling the lifecycle.
For that I used a small trick using a fragment to get a hook of the activity lifecycle events and start/stop the
location and sensor updates appropriately.

## The code
The code is published in [gist][the-gist] and available under [Apache 2 License][license]

[google-maps]: https://developers.google.com/maps/documentation/android-api
[set-my-location-enabled]: https://developers.google.com/android/reference/com/google/android/gms/maps/GoogleMap#setMyLocationEnabled(boolean)
[location-source]: https://developers.google.com/android/reference/com/google/android/gms/maps/LocationSource
[location-tutorial]: http://developer.android.com/training/location/receive-location-updates.html
[smooth-article]: http://www.ymc.ch/en/smooth-true-north-compass-values
[the-gist]: https://gist.github.com/groupsky/8772ba063957e7ace78641909b8c6c35
[license]: http://www.apache.org/licenses/LICENSE-2.0
