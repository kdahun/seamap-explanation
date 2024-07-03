# seamap-explanation
---

---
## MainActivity2.java

```
// 해구 번호, 위도, 경도, 파고를 저장클래스
class GridData{
    int gridNumber; // 해구 번호
    double latitude; // 위도
    double lonitude; // 경도
    double wave; // 파고 크기

    // 생성자
    public GridData(int gridNumber, double latitude, double lonitude, double wave){
        this.gridNumber = gridNumber;
        this.latitude = latitude;
        this.lonitude = lonitude;
        this.wave = wave;
    }
}
```

```
class GridDataManager{
    public List<GridData> gridDataList; // Arraylist 만들기

    // 생성자
    public GridDataManager(){
        gridDataList = new ArrayList<>();
    }

    // 위도와 경도를 arraylist에 추가해줌
    void addGrid(int number, double lat, double lng,double wave){
        gridDataList.add(new GridData(number,lat,lng,wave));
    }
}
```

```
public class MainActivity2 extends AppCompatActivity implements OnMapReadyCallback {

    RequestQueue queue; // volley 라이브러리의 구성요소
    ArrayList<Double> lati; // 위도
    ArrayList<Double> longti; // 경도
    ArrayList<Double> waveSize; // 파고

    GoogleMap gMap;
    MapFragment mapFrag;
    // 그려진 경로를 나타내는 폴리라인 객체를 보관, 파고를 나타내는 폴리콘 객체를 보관
    private List<Polyline> shortestPathPolylines = new ArrayList<>();
    private List<Polygon> pagopolygons = new ArrayList<>();

    GridDataManager manager = new GridDataManager();

    // 상태 추적을 위한 변수 추가
    private boolean isGridPolygonDrawn = false;
    private boolean isShortestPathShown = false;

    private double message;

    double Current_lat;
    double Current_log;
    private String url;
    
    // 서브 메뉴 만들기
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        super.onCreateOptionsMenu(menu);
        menu.add(0,1,0,"실시간 파고 On/Off");
        menu.add(0,2,0,"실시간 경로 On/Off");
        SubMenu subMenu = menu.addSubMenu("예측 파고 >>");
        ...
        return true;
    }

// 메뉴 클릭 이벤트
    @Override
    public boolean onOptionsItemSelected(@NonNull MenuItem item) {
        switch(item.getItemId()){
            case 1:
                url = "http://202.31.147.129:25003/weater.php"; // php 파싱을 위한 url 주소
                // 실시간 모드
                if (isGridPolygonDrawn) { // 만약 isGridPolygonDrawn이 true이면 Polygon을 삭제
                    for (Polygon Polygon : pagopolygons) {
                        Polygon.remove();
                    }
                    isGridPolygonDrawn = false;
                } else { // false이면 getData() 메서드를 통해 파고의 위도, 경도, 상태를 php 파싱을 통해 polygon을 그려준다.

                    manager = new GridDataManager();
                    getData(url); // polygon을 그려주는 곳은 drawPolygon에서 그려준다.
                    isGridPolygonDrawn = true;
                }
                break;

            case 2:
                // 경로 예측 모드
                if (isShortestPathShown) { // isShortestPathShown이 true이면 polyline 삭제
                    for (Polyline polyline : shortestPathPolylines) {
                        polyline.remove();
                    }
                    isShortestPathShown = false;
                } else {    // false이면 최단경로 그려주기
                    getShortestPathData(); // 최단 경로 데이터 가져오기
                    isShortestPathShown = true;
                }
                break;

            ...

        }

        return true;
    }
    // =======================================데이터를 가져오는 부분===================================================
    private void getData(String url) {
        lati = new ArrayList<Double>();                             // 위도 저장
        longti = new ArrayList<Double>();                           // 경도 저장
        waveSize = new ArrayList<Double>();                         // 파도 저장

        if (queue == null) {
            queue = Volley.newRequestQueue(this);
        }

        // php 파싱을 통해 Json 파싱을 해 위도, 경도, 파고 저장
        StringRequest stringRequest = new StringRequest(Request.Method.GET, url, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                try {
                    JSONObject jsonObject = new JSONObject(response);
                    JSONArray jsonArray = jsonObject.getJSONArray("blackice");

                    for (int i = 0; i < jsonArray.length(); i++) {
                        JSONObject item = jsonArray.getJSONObject(i);

                        // 위도 저장
                        double latitude = Double.parseDouble(item.getString("latitude"));

                        // 경도 저장
                        double longtitude = Double.parseDouble(item.getString("longitude"));

                        // 파고 저장
                        String waveS = item.getString("wavesize");
                        String resultWave = waveS.substring(0, waveS.length() - 1);
                        double dWaveSize;

                        //파고가 널값이면 0을 넣어준다.
                        try {
                            dWaveSize = Double.parseDouble(resultWave);
                        } catch (Exception e) {
                            dWaveSize = 0;
                        }

                        // 위도, 경도, 파고 arraylist에 저장
                        lati.add(latitude);
                        longti.add(longtitude);
                        waveSize.add(dWaveSize);
                    }

                    // 데이터가 잘 가져와졌으면 "데이터 로드 완료"  메시지 toast 
                    for (int i = 0; i < lati.size(); i++) {
                        // manager에 저장.
                        manager.addGrid(i,lati.get(i), longti.get(i),waveSize.get(i));
                    }
                    // Polygon 그려주기
                    drawGridPolygon(gMap, manager); // Call the new method

                    Toast.makeText(getApplicationContext(), "데이터 로드 완료", Toast.LENGTH_SHORT).show();

                    onDataLodaed();// 데이터 로드가 끝난후에

                } catch (JSONException e) {
                    e.printStackTrace();
                }
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                //에러
                Toast.makeText(getApplicationContext(), "오류발생", Toast.LENGTH_SHORT).show();
            }
        });
        queue.add(stringRequest);
    }
//========================================================================================================================

    // 파고 높이별로 색 지정(drawGridPolygon에서 파고의 Polygon에 색 넣어주기
    private int calculateColorBaseOnDepth(double depth) {
        int brightness = 0;
        if (depth == 0) {
            brightness = 0;
        } else if (depth < 1.5) {
            brightness = 255 / 6;
        } else if (depth < 2) {
            brightness = 255 / 6 * 2;
        } else if (depth < 2.5) {
            brightness = 255 / 6 * 3;
        } else if (depth < 3) {
            brightness = 255 / 6 * 4;
        } else if (depth < 3.5) {
            brightness = 255 / 6 * 5;
        } else if (depth >= 3.5) {
            brightness = 254;
        }
        // Color.rgb에 색 밝기값을 넣어서 반환
        return brightness;
    }
    // 위도 경도가 0.5차이로 있기 때문에 0.25차이로 Polygon을 그리기
    private PolygonOptions createPolygonForCoordinate(double latCenter, double lngCenter){
        PolygonOptions rectOptions=new PolygonOptions()
                .add(new LatLng(latCenter - 0.25 , lngCenter - 0.25))
                .add(new LatLng(latCenter - 0.25 , lngCenter + 0.25))
                .add(new LatLng(latCenter + 0.25 , lngCenter + 0.25))
                .add(new LatLng(latCenter + 0.25 , lngCenter - 0.25))
                .strokeWidth(2)
                .strokeColor(Color.BLACK);

        return rectOptions;
    }

    // Googloe 지도에 그리드 폴리곤(다각형)을 그리는 기능을 담당한다.
    public void drawGridPolygon(GoogleMap googleMap, GridDataManager manager) {

        // Polygon이 있으면 삭제해주기
        for (Polygon polygon : pagopolygons) {
            // 폴리곤을 제거
            polygon.remove();
        }
        // 리스트를 비워준다.
        pagopolygons.clear();

        // 새로운 polygons 그리기
        for (GridData data : manager.gridDataList) {
            // createPolygonForCoordinate를 통해 Polygon 생성해 변수에 저장
            PolygonOptions polygonOptions = createPolygonForCoordinate(data.latitude, data.lonitude);

            // 파고에 따른 색
            int b = calculateColorBaseOnDepth(data.wave);
            int fillColor = Color.argb(178 , b, 0, 0); // (투명도, red, green, blue)

            polygonOptions.fillColor(fillColor); // polygon에 색 넣기

            // googlemap에 polygon 생성
            Polygon polygon = googleMap.addPolygon(polygonOptions);
            pagopolygons.add(polygon);
        }
    }
//======여기까지가 실시간 파고 그려주기============

//============ 여기에서 부터 경로 그려주기 ===================
    private void getShortestPathData() { // php 파싱을 통해 python의 networkx로 만든 최단 경로 가져오기
        String url = "http://202.31.147.129:25003/shortest.php";

        if (queue == null) {
            queue = Volley.newRequestQueue(this);
        }
        // 서버에서 최단 경로 데이터 가져오기
        StringRequest stringRequest = new StringRequest(Request.Method.GET, url, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                try {
                    JSONObject jsonObject = new JSONObject(response);
                    JSONArray jsonArray = jsonObject.getJSONArray("shortest");

                    // 최단 경로 위도 경도를 ArrayList에 저장
                    ArrayList<LatLng> latLngs = new ArrayList<>();

                    for (int i = 0; i < jsonArray.length(); i++) {
                        JSONObject item = jsonArray.getJSONObject(i);

                        // 위도 저장
                        double latitude = Double.parseDouble(item.getString("latitude"));

                        // 경도 저장
                        double longitude = Double.parseDouble(item.getString("longitude"));

                        message = Double.parseDouble(item.getString("message"));

                        latLngs.add(new LatLng(latitude, longitude));
                    }
                    if (message == 1) {
                        // 이 부분에서 최단 경로 찍어줌.
                        Toast.makeText(getApplicationContext(), "최적 향로 가져오기 완료", Toast.LENGTH_LONG).show();
                        // 최단 경로를 그려주는 메서드
                        drawShortestPath(latLngs);
                    } else if (message == 2) {
                        // 만약 갈 수 있는 길이 없을 경우
                        Toast.makeText(getApplicationContext(), "기상 악화로 인한 향로 불안정", Toast.LENGTH_LONG).show();
                    } else if (message == 3) {
                        // 목적지를 이상한 곳에 찍은 경우
                        Toast.makeText(getApplicationContext(), "잘못된 목적지 설정", Toast.LENGTH_LONG).show();
                    }
                } catch (JSONException e) {
                    e.printStackTrace();
                }
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                Toast.makeText(getApplicationContext(), "오류발생", Toast.LENGTH_SHORT).show();
            }
        });
        queue.add(stringRequest);
    }

    // 추가: 최단 경로 선 그리기 함수
    private void drawShortestPath(ArrayList<LatLng> latLngs) {
        if (gMap != null && latLngs != null && !latLngs.isEmpty()) {
            // 기존 경로 Polyline 객체들을 모두 제거합니다.
            for (Polyline polyline : shortestPathPolylines) {
                polyline.remove();
            }
            shortestPathPolylines.clear();

            // for 문으로 점과 점을 이어주기 위해 
            for (int i = 0; i < latLngs.size() - 1; i++) {
                LatLng start = latLngs.get(i);
                LatLng end = latLngs.get(i + 1);

                Polyline polyline = gMap.addPolyline(new PolylineOptions()
                        .add(start, end)
                        .width(10)
                        .color(Color.BLUE)
                );

                shortestPathPolylines.add(polyline); // 리스트에 추가
            }
        }
    }

    // 권한 요청을 식별하기 위한 고유한 코드. 요청과 응답을 매칭하는 데 사용
    // 일반적으로 임의의 정수로 정의되며 여기서는 1로 설정되어 있다
    private static final int LOCATION_PERMISSION_REQUEST_CODE = 1;

    // 비동기적으로 지도 초기화
    private void onDataLodaed() {
        mapFrag = (MapFragment) getFragmentManager().findFragmentById(R.id.map);
        mapFrag.getMapAsync(this);
    }

    private void checkLocationPermission() { // 위치 권한을 확인하고 필요한 경우 요청하는 역할
        // 위치 권한 확인
        // ContextCompat.checkSelfPermission은 주어진 권한이 앱에 부여되었는지 확인
        // Manifest.permission.ACCESS_FINE_LOCATION은 정밀한 위치 정보를 액세스할 수 있는 권한을 의미한다.
        // PackageManager.PERMISSION_GRANTED와 비교하여 권한이 부여되지 않았을 경우 확인
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) { // 권한이 없을 경우
            
            // 사용자가 이전에 권한 요청을 거부했는지 확인
            if (ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.ACCESS_FINE_LOCATION)) {
                // 필요한 권한 설명을 위한 다이얼 로그 표시
            } else {
                // 사용자가 이전에 요청을 거부하지 않았거나 처음 요청하는 경우
                // 권한 요청
                ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.ACCESS_FINE_LOCATION}, LOCATION_PERMISSION_REQUEST_CODE);
            }
        } else {
            // 권한이 이미 있는 경우
            // 위치 정보 가져오기 진행
            onDataLodaed();
        }
    }
// 데이터 POST 메서드
    private void sendData() {
        // 서버 URL 설정
        String url = "http://202.31.147.129:25003/get.php";

        //사용자 디바이스 고유ID 가져오는 코드
        String deviceId = Settings.Secure.getString(getContentResolver(), Settings.Secure.ANDROID_ID);

        // Volley 큐 초기화
        if (queue == null) {
            queue = Volley.newRequestQueue(this);
        }

        // 서버에 POST 요청을 보낼 StringRequest 생성
        StringRequest stringRequest = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                // 서버에서 전송된 결과를 처리할 코드
                Toast.makeText(getApplicationContext(), "데이터 전송 완료(sendData)", Toast.LENGTH_SHORT).show();
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                // 에러 발생 시 처리할 코드
                Toast.makeText(getApplicationContext(), "sendData 오류발생", Toast.LENGTH_SHORT).show();
            }
        }) {
            @Override
            protected Map<String, String> getParams() throws AuthFailureError {
                // MainActivity에서 전달 받은 목적지 좌표 정보 가져오기
                Intent intent = getIntent();
                //Toast.makeText(getApplicationContext(),intent.getStringExtra("destination_lat"),Toast.LENGTH_SHORT).show();
                String destination_lat = intent.getStringExtra("destination_lat");
                String destination_log = intent.getStringExtra("destination_log");


                Map<String, String> params = new HashMap<>();
                params.put("dest_latitude", String.valueOf(destination_lat)); // 목적지 경도
                params.put("dest_longitude", String.valueOf(destination_log)); // 목적지 위도
                params.put("cur_latitude", String.valueOf(Current_lat)); // 현재 경도
                params.put("cur_longitude", String.valueOf(Current_log)); // 현재 위도
                params.put("device_id", deviceId);
                return params;
            }
        };

        queue.add(stringRequest);
    }

    // 현재 위치 정보 제공 클라이언트
    private FusedLocationProviderClient fusedLocationProviderClient;

    // 현재 위치 정보를 저장할 변수
    private LatLng currentLocation;


    // 구글맵 셋팅?
    // onMapReady 메서드는 Google Maps가 준비가 되었을 때 호출되며, 이를 통해 개발자가 지도를 커스터마이징하고 지도 작업을 수행할 수 있도록 해준다.
    @Override
    public void onMapReady(@NonNull GoogleMap googleMap) {
        gMap = googleMap;
        gMap.setMapType(GoogleMap.MAP_TYPE_NORMAL); // Map type

        LatLng korea = new LatLng(37, 128); // 지도 위치 설정
        gMap.moveCamera(CameraUpdateFactory.newLatLngZoom(korea, 5)); // 카메라 위치 변경 및 줌 어느정도

        fusedLocationProviderClient = LocationServices.getFusedLocationProviderClient(this);

        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED && ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            return;
        }
    }

    // LocationRequest : 위치 요청을 정의하는 클래스 위치 요청의 간격, 우선순위 등을 설정할 수 있다.
    private LocationRequest locationRequest;
    private LocationCallback locationCallback;

    // 현재 위치 업데이트 요청 설정
    private void setupLoctionUpdate() {
        locationRequest = LocationRequest.create();
        locationRequest.setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY);
        locationRequest.setInterval(10000); // 위치 업데이트 간격(10초)

        locationCallback = new LocationCallback() {
            @Override
            public void onLocationResult(@NonNull LocationResult locationResult) {
                if (locationResult != null) {
                    for (android.location.Location location : locationResult.getLocations()) {

                        // 현재 위치의 경위도 값을 변수에 저장
                        Current_lat = location.getLatitude();
                        Current_log = location.getLongitude();
                        sendData();

                        //Toast.makeText(getApplicationContext(),Current_lat+":"+Current_log,Toast.LENGTH_SHORT).show();
                        // 새로운 위치를 받아서 처리
                        updateCurrentLocation(location);
                    }
                }
            }
        };
    }

private Marker currentLoactionMarker; // 현재 위치를 나타내는 마커

    private Circle currentCircle;

    private void updateCurrentLocation(android.location.Location location) {
        // 새로운 위치 업데이트 시, 지도에 마커 표시
        if (gMap != null) {
            LatLng newLatLng = new LatLng(location.getLatitude(), location.getLongitude());
            if(currentLoactionMarker == null){
                // 초기에 마커가 없다면 추가
                currentLoactionMarker = gMap.addMarker(new MarkerOptions().position(newLatLng).zIndex(2.0f));
            }else{
                // 이미 마커가 있다면 위치만 업데이트
                currentLoactionMarker.setPosition(newLatLng);
            }

            // 반경 원을 그리기 위한 설정
            CircleOptions circleOptions = new CircleOptions().
                    center(newLatLng).
                    radius(50000).
                    strokeColor(Color.BLUE).
                    fillColor(Color.TRANSPARENT).zIndex(1.0f);

            // 이전에 추가된 반경원이 있다면 제거
            if(currentCircle != null){
                currentCircle.remove();
            }

            // 새로운 반경원을 지도에 추가
            currentCircle = gMap.addCircle(circleOptions);

            // 위치 업데이트 발생 시 Toast 메시지 표시
            //Toast.makeText(this,"현재 위치가 업데이트 되었습니다",Toast.LENGTH_SHORT).show();
        }
    }
}
```
queue : Volley 라이브러리에서 제공하는 RequestQueue 클래스의 인스턴스이다.
- 네트워크 요청 관리 : RequestQueue는 네트워크 요청을 관리하고 처리하기 위해 설계된 Volley 라이브러리의 구성요소이다. 이 큐는 요청을 추적하고 순서대로 실행되도록 보장한다. 이는 너무 많은 요청이 한 번에 보내지는 것을 방지하여 네트워크 또는 서버가 과부화되지 않도록 한다.
- 다중 요청 처리 : RequestQueue는 네트워크 요청을 처리하는 작업 관리자처럼 동작한다. 여러 요청을 큐에 추가할 수 있으며, 순차적으로 또는 동시다발적으로 처리할 수 있다. 이는 여러 네트워크 작업이 있을 때 효율적으로 관리하는 데 도움이 된다.
- 재시도 정책 및 우선 순위 : Volley는 RequestQueue를 사용하여 재시도 정책 및 요청의 우선순위를 관리한다. 요청이 실패하면, RequestQueue는 사전 정의된 정책에 따라 요청을 다시 시도할 수 있다. 요청은 또한 중요도에 따라 우선순위를 설정하여 중요한 요청이 먼저 처리되도록 할 수 있다.
- 취소 및 생명 주기 관리 : 필요하지 않은 경우 요청을 취소할 수 있으며, RequestQueue는 이러한 요청의 생명주기를 관리하는 데 도움을 준다. 이는 불필요한 네트워크 작업을 피하고 자원사용을 관리하는 데 유용하다.

### getData메서드에서 queue가 사용되는 이유
1. 요청 큐 초기화
```
if(queue == null){
  queue = Volley.newRequestQueue(this);
}
```
이는 RequestQueue가 한 번만 초기화되도록 보장한다. queue가 이미 초기화된 경우 새로 생성하지 않음으로써 불필요한 자원 할당을 방지한다.

2. 네트워크 요청 추가
```
queue.add(stringRequest);
```
이 코드는 StringRequest를 RequestQueue에 추가한다. 요청을 큐에 추가하면 이 네트워크 작업의 처리를 Volley 라이브러리에 위임하게 된다. Volley는 요청을 처리하고, 네트워크 작업을 관리하며, 응답을 구문 분석하고, 요청에서 정의된 적절한 콜백 메서드(onResponse 또는 onErrorResponse)를 관리한다.

### queue가 전체 아키텍처에서 어떻게 작동하는 가
1. 요청 실행
    - 요청을 queue에 추가하면, Volley는 해당 요청의 실행을 관리한다.
    - 여기에는 지정된 URL로 요청을 보내고 응답을 기다린 후 적절한 콜백 메서드를 호출하는 작업이 포함된다.
      
2. 응답 처리
    - 서버가 응답을 보내면 Volley는 RequestQueue를 사용하여 응답을 올바른 핸들러로 전달한다.
    - 이 경우, 응답은 onResponse메서드에서 처리되어 JSON 데이터를 구문 분석하고 처리한다.
  
3. 스레드 관리
    - RequestQueue는 네트워크 작업을 위한 스레드를 관리한다.
    - 네트워크 작업은 별도의 스레드에서 실행되어, 메인 스레드가 차단되지 않도록 하여, 애플리케이션의 사용자 인터페이스가 부드럽고 반응성을 유지할 수 있도록 한다.
      
4. 싱글턴 패턴
    - 종종 RequestQueue는 애플리케이션의 모든 네트워크 요청을 관리하는 단일 인스턴스만 존재하도록 싱글턴 패턴으로 구현한다.
    - 이는 명시적으로 나타나지는 않지만, queue가 null인 경우에만 초기화하는 것은 이러한 접근 방식을 따르는 것이다.
      
 => 요약하자면 queue는 네트워크 작업을 효율적으로 처리하는 데 필수적이다. 네트워크 관리, 재시도, 스레드 관리 및 요청 우선순위를 추상화한다. RequestQueue를 사용하면 이러한 책임을 Volley 라이브러리에 맡길 수 있어, 애플리케이션의 비즈니스 로직과 데이터 처리에 집중할 수 있다.

### sendData() 메서드 추가 설명
- Volley : 구글에서 제공하는 HTTP라이브러리로, 간단한 API로 네트워크 요청을 관리한다. RequestQueue를 사용하여 요청을 관리하고, StringRequest나 JosnObjectRequest 등 다양한 요청 타입을 지원한다.
- Android ID : 안드로이드 디바이스에 고유한 ID로, 각 디바이스에 고유한 값이다. 이는 앱 디바이스를 식별하는 데 유용하다.
- Intent : 안드로이드의 Intent는 다른 액티비티에서 전달된 데이터를 받아오는 데 사용된다.


### onMapReady() 설명
- onMapReady 메서드는 Google Maps가 준비되었을 때 호출된다. 이 시점에서 개발자는 지도를 커스터마이징하거나 작업을 수행할 수 있다.
- @override : onMapReady 메서드가 GoogleMap.OnMapReadycallback 인터페이스의 메서드를 재정의하고 있다는 것을 나타낸다.
- @NonNull : 이 어노테이션은 googleMap 매개변수가 null이 될 수 없음을 나타낸다.
- fusedLocationProviderClient = LocationServices.getFusedLocationProviderClient(this); : 위치 서비스를 관리한다. 이 클라이언트는 GPS, Wi-Fi, 셀룰러 데이터 등 여러 위치 소스를 통합하여 보다 정확한 위치 정보를 제공한다.
```
- if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED && ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
    return;
}
```
- 권한 확인 : ACCESS_FINE_LOCATION과 ACCESS_COARSE_LOCATION 두 가지 위치 접근 권한이 있는지 확인한다.
- ActivityCompat.checkSelfPermission : 현재 권한이 있는지 확인하는 메서드
  * Manifest.permission.ACCESS_FINE_LOCATION : 고정밀 위치 접근 권한
  * Manifest.permission.ACCESS_COARSE_LOCATION : 저정밀 위치 접근 권한



