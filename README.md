
    private static final long GEO_DURATION = 60 * 60 * 1000;
    //    private static final String GEOFENCE_REQ_ID = "My Geofence";
    private static final float GEOFENCE_RADIUS = 500.0f; // in meters
    private PendingIntent geoFencePendingIntent;
    private final int GEOFENCE_REQ_CODE = 0;
    private Circle geoFenceLimits;
    private ArrayList<LatLng> clRoutePlanLatLng;
    private ArrayList<Circle> clGeoFenceLimitsList;
    private ArrayList<Marker> clRoutePlanMarkersList;


// adding geo button  


  private View getGeoBtnLayout() {
        LinearLayout clLayout = new LinearLayout(clContext);
        clLayout.setOrientation(LinearLayout.VERTICAL);
        clLayout.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT));

        Button clButton = new Button(clContext);
        clLayout.addView(clButton);
        clButton.setText("addGeo");
        clButton.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT));
        clButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                addGeoFenceForRoutePlanData(0);
            }
        });

        Button clButton2 = new Button(clContext);
        clLayout.addView(clButton2);
        clButton2.setText("remove geo");
        clButton2.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT));
        clButton2.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                clearGeofence();
            }
        });
        return clLayout;
    }


// remove geofence

// Clear Geofence
    private void clearGeofence() {
        Log.d(TAG, "clearGeofence()");
        if (clGoogleApiClient != null) {

            LocationServices.GeofencingApi.removeGeofences(
                    clGoogleApiClient,
                    createGeofencePendingIntent()
            ).setResultCallback(new ResultCallback<Status>() {
                @Override
                public void onResult(@NonNull Status status) {
                    if (status.isSuccess()) {
                        // remove drawing
                        removeGeofenceDraw();
                    }
                }
            });
        }
    }


 // Draw Geofence circle on GoogleMap
    private void drawGeofence(ArrayList<LatLng> clLatLng) {
        Log.d(TAG, "drawGeofence()");

        clGeoFenceLimitsList = new ArrayList<>();
        for (int i = 0; clLatLng != null && i < clLatLng.size(); i++) {
            CircleOptions circleOptions = new CircleOptions()
                    .center(clLatLng.get(i))
                    .strokeColor(Color.argb(50, 70, 70, 70))
                    .fillColor(Color.argb(100, 150, 150, 150))
                    .radius(GEOFENCE_RADIUS);
            geoFenceLimits = googleMap.addCircle(circleOptions);
            clGeoFenceLimitsList.add(geoFenceLimits);
        }
    }


 private PendingIntent createGeofencePendingIntent() {
        Log.d(TAG, "createGeofencePendingIntent");

        if (geoFencePendingIntent != null)
            return geoFencePendingIntent;
        Intent intent = new Intent(this, GeofenceTrasitionService.class);
        return PendingIntent.getService(
                this, GEOFENCE_REQ_CODE, intent, PendingIntent.FLAG_UPDATE_CURRENT);
    }


    private static final String NOTIFICATION_MSG = "NOTIFICATION MSG";

    // Create a Intent send by the notification
    public static Intent makeNotificationIntent(Context context, String msg) {
        Intent intent = new Intent(context, CLHomeActivity.class);
        intent.putExtra(NOTIFICATION_MSG, msg);
        return intent;
    }

    // Create a Geofence
    private Geofence createGeofence(LatLng latLng, float radius) {
        Log.d(TAG, "createGeofence");
        String GEOFENCE_REQ_ID = UUID.randomUUID().toString();
        return new Geofence.Builder()
                .setRequestId(GEOFENCE_REQ_ID)
                .setCircularRegion(latLng.latitude, latLng.longitude, radius)
                .setExpirationDuration(Geofence.NEVER_EXPIRE)
                .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER
                        | Geofence.GEOFENCE_TRANSITION_EXIT)
                .build();
    }

    // Create a Geofence Request
    private GeofencingRequest createGeofenceRequest(List<Geofence> clGeofenceList) {
        Log.d(TAG, "createGeofenceRequest");
        return new GeofencingRequest.Builder()
                .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
//                .addGeofence(geofence)
                .addGeofences(clGeofenceList)
                .build();
    }

    // Add the created GeofenceRequest to the device's monitoring list
    private void addGeofence(GeofencingRequest request) {
        Log.d(TAG, "addGeofence");
        if (clGoogleApiClient != null) {

            if (ActivityCompat.checkSelfPermission(this, android.Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
                return;
            }
            LocationServices.GeofencingApi.addGeofences(
                    clGoogleApiClient,
                    request,
                    createGeofencePendingIntent()
            ).setResultCallback(new ResultCallback<Status>() {
                @Override
                public void onResult(@NonNull Status status) {
                    Log.i(TAG, "onResult: " + status);
                    if (status.isSuccess()) {
                        Toast.makeText(clContext, "" + status, Toast.LENGTH_SHORT).show();
                        drawGeofence(clRoutePlanLatLng);

                    } else {
                        // inform about fail
                        Toast.makeText(clContext, "unable to add geo fence", Toast.LENGTH_SHORT).show();
                    }

                }
            });
        }
    }

private void addGeoFenceForRoutePlanData(int iNoOfDays) {
        List<Geofence> clGeofenceList = new ArrayList<>();
        if (iNoOfDays > -1) {
            ArrayList<CLLocationDTO> alLocationDTO = CLDBStore.getInstance(clContext).getRoutePlanDetailsDateWise(CLTrackerAppContext.getInstance().getUserId(), getIntDate(iNoOfDays));
            if (alLocationDTO != null) {

                clRoutePlanMarkersList = new ArrayList<>();
                for (int i = 0; i < alLocationDTO.size(); i++) {
                    CLLocationDTO clLocationDTO = alLocationDTO.get(i);
                    double clLatitude = clLocationDTO.getLatitude().doubleValue();
                    double clLongitude = clLocationDTO.getLongitude().doubleValue();
                    LatLng clLatLng = new LatLng(clLatitude, clLongitude);
                    if (clRoutePlanLatLng == null)
                        clRoutePlanLatLng = new ArrayList<>();
                    if (alLocationDTO.get(i).getAccountName().equals("Agent (David Warui Wathenge) Malava")) {
                        clRoutePlanLatLng.add(clLatLng);
                        if (googleMap != null) {
                            Marker clMarker = googleMap.addMarker(new MarkerOptions().position(clLatLng).icon(BitmapDescriptorFactory.defaultMarker(BitmapDescriptorFactory.HUE_GREEN)));
                            clRoutePlanMarkersList.add(clMarker);
                        }
                    }
                    clGeofenceList.add(createGeofence(clLatLng, GEOFENCE_RADIUS));

                }
                if (clGeofenceList.size() > 0) {
                    GeofencingRequest clGeofencingRequest = createGeofenceRequest(clGeofenceList);
                    addGeofence(clGeofencingRequest);
                }
            }
            //Object[] arrStartingPoints = CLDBStore.getInstance(clContext).getUserStartingPoints(CLTrackerAppContext.getInstance().getUserId());
        }
    }

private void removeGeofenceDraw() {
        Log.d(TAG, "removeGeofenceDraw()");
        if (clRoutePlanMarkersList != null && clGeoFenceLimitsList != null)
            for (int i = 0; i < clRoutePlanMarkersList.size(); i++)
                clRoutePlanMarkersList.get(i).remove();

        if (clGeoFenceLimitsList != null)
            for (int i = 0; i < clGeoFenceLimitsList.size(); i++)
                clGeoFenceLimitsList.get(i).remove();
    }


