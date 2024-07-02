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
