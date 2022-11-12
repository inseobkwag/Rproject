# 602277101 곽인섭

11월9일 11주차
==================
<b>7.2 요즘 뜨는 지역은 어디?</b>

2-1: 데이터 준비
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) 
load("./06_geodataframe/06_apt_price.rdata")     
grid <- st_read("./01_code/sigun_grid/seoul.shp") 
apt_price <-st_join(apt_price, grid, join = st_intersects)  
head(apt_price, 2)
```
2-2: 이전/이후 데이터세트 생성
```
kde_before <- subset(apt_price, ymd < "2021-07-01")  
kde_before <- aggregate(kde_before$py, by=list(kde_before$ID),mean)  
colnames(kde_before) <- c("ID", "before")   

kde_after  <- subset(apt_price, ymd > "2021-07-01")  
kde_after <- aggregate(kde_after$py, by=list(kde_after$ID),mean)  
colnames(kde_after) <- c("ID", "after")  

kde_diff <- merge(kde_before, kde_after, by="ID")    
kde_diff$diff <- round((((kde_diff$after-kde_diff$before)/
                           kde_diff$before)* 100), 0) 
head(kde_diff, 2)
```
2-3: 가격이 오른지역 찾기
```
library(sf)        
kde_diff <- kde_diff[kde_diff$diff > 0,]    
kde_hot <- merge(grid, kde_diff,  by="ID")  
library(ggplot2)   
library(dplyr)     
kde_hot %>%        
  ggplot(aes(fill = diff)) + 
  geom_sf() + 
  scale_fill_gradient(low = "white", high = "red")
```
2-4: 지도경계선 그리기
```
library(sp)   
kde_hot_sp <- as(st_geometry(kde_hot), "Spatial") 
x <- coordinates(kde_hot_sp)[,1]  
y <- coordinates(kde_hot_sp)[,2] 

l1 <- bbox(kde_hot_sp)[1,1] - (bbox(kde_hot_sp)[1,1]*0.0001) 
l2 <- bbox(kde_hot_sp)[1,2] + (bbox(kde_hot_sp)[1,2]*0.0001)
l3 <- bbox(kde_hot_sp)[2,1] - (bbox(kde_hot_sp)[2,1]*0.0001)
l4 <- bbox(kde_hot_sp)[2,2] + (bbox(kde_hot_sp)[1,1]*0.0001)

library(spatstat)  
win <- owin(xrange=c(l1,l2), yrange=c(l3,l4))  
plot(win)                                     
rm(list = c("kde_hot_sp", "apt_price", "l1", "l2", "l3", "l4"))
```
2-5: 밀도 그래프 변환
* 경계창 위에 좌표값 포인트 생성
* 포인트를 커널밀도 함수로 변환
```
p <- ppp(x, y, window=win, marks=kde_hot$diff) 
d <- density.ppp(p, weights=kde_hot$diff,      
                 sigma = bw.diggle(p), 
                 kernel = 'gaussian')
plot(d)   
rm(list = c("x", "y", "win","p"))
```
2-6: 픽셀을 레스터로 변환
* 노이즈 제거
* install.packages("raster")
```
d[d < quantile(d)[4] + (quantile(d)[4]*0.1)] <- NA  
library(raster)         
raster_hot <- raster(d) 
plot(raster_hot)
```
2-7: 클리핑
* 서울시 경계선 불러오기
* 외곽선 클리핑
* 좌표계 정의
*  확인
```
bnd <- st_read("./01_code/sigun_bnd/seoul.shp") 
raster_hot <- crop(raster_hot, extent(bnd))            
crs(raster_hot) <- sp::CRS("+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84  
                        +towgs84=0,0,0")  
plot(raster_hot)   
plot(bnd, col=NA, border = "red", add=TRUE)
```
2-8: 지도 그리기
* 베이스맵 불러오기
* 서울시 경계선 불러오기
* 레스터 이미지 불러오기
```
library(leaflet)   # install.packages("leaflet")
leaflet() %>%
  addProviderTiles(providers$CartoDB.Positron) %>% 
  addPolygons(data = bnd, weight = 3, color= "red", fill = NA) %>% 
  addRasterImage(raster_hot, 
   colors = colorNumeric(c("blue", "green", "yellow","red"), 
   values(raster_hot), na.color = "transparent"), opacity = 0.4)
```
2-9: 평균 가격 변화율 정보 저장,메모리 정리
```
save(raster_hot, file="./07_map/07_kde_hot.rdata") 
rm(list = ls())
```
<b>7.3 우리동네가 옆 동네보다 비쌀까?</b>

3-1: 데이터 준비
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) 
load("./06_geodataframe/06_apt_price.rdata")   
load("./07_map/07_kde_high.rdata")    
load("./07_map/07_kde_hot.rdata")     

library(sf)    
bnd <- st_read("./01_code/sigun_bnd/seoul.shp")   
grid <- st_read("./01_code/sigun_grid/seoul.shp")
```
3-2: 마커 클러스터링 옵션 설정
* 마커 클러스터링 함수 등록
* 마커 클러스터링 컬러 설정: 상, 중, 하
```
pcnt_10 <- as.numeric(quantile(apt_price$py, probs = seq(.1, .9, by = .1))[1])
pcnt_90 <- as.numeric(quantile(apt_price$py, probs = seq(.1, .9, by = .1))[9])
load("./01_code/circle_marker/circle_marker.rdata")
circle.colors <- sample(x=c("red","green","blue"),size=1000, replace=TRUE)
```
3-3: 마커 클러스터링 시각화
* install.packages("purrr")
* 오픈스트리트맵 불러오기
* 서울시 경계선 불러오기
* 최고가 레스터 이미지 불러오기
* 급등지 레스터 이미지 불러오기
* 최고가 / 급등지 선택 옵션 추가하기
* 마커 클러스터링 불러오기
* 메모리 정리하기
```
library(purrr)  
leaflet() %>% 
  addTiles() %>%  
  addPolygons(data = bnd, weight = 3, color= "red", fill = NA) %>%
  addRasterImage(raster_high, 
    colors = colorNumeric(c("blue","green","yellow","red"), values(raster_high), 
    na.color = "transparent"), opacity = 0.4, group = "2021 최고가") %>% 
  addRasterImage(raster_hot, 
    colors = colorNumeric(c("blue", "green", "yellow","red"), values(raster_hot), 
    na.color = "transparent"), opacity = 0.4, group = "2021 급등지") %>%   
  addLayersControl(baseGroups = c("2021 최고가", "2021 급등지"), options = layersControlOptions(collapsed = FALSE)) %>%
  addCircleMarkers(data = apt_price, lng =unlist(map(apt_price$geometry,1)), 
                   lat = unlist(map(apt_price$geometry,2)), radius = 10, stroke = FALSE, 
                   fillOpacity = 0.6, fillColor = circle.colors, weight=apt_price$py, 
                   clusterOptions = markerClusterOptions(iconCreateFunction=JS(avg.formula))) 
rm(list = ls())
```

11월2일 10주차
==================
7.분석주제를 지도로 시각화

1-1: 지역별 평균 가격구하기
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) 
load("./06_geodataframe/06_apt_price.rdata")   
library(sf)     
grid <- st_read("./01_code/sigun_grid/seoul.shp")     
apt_price <-st_join(apt_price, grid, join = st_intersects)  
head(apt_price)

kde_high <- aggregate(apt_price$py, by=list(apt_price$ID), mean) 
colnames(kde_high) <- c("ID", "avg_price")   
head(kde_high, 2)
```
1-2: 평균가격 정보표시
```
kde_high <- merge(grid, kde_high,  by="ID") 
library(ggplot2) 
library(dplyr)   
kde_high %>% ggplot(aes(fill = avg_price)) + 
  geom_sf() + 
  scale_fill_gradient(low = "white", high = "red")
```
1-3: 지도 경계 그리기
```
library(sp) 
kde_high_sp <- as(st_geometry(kde_high), "Spatial")    
x <- coordinates(kde_high_sp)[,1]  
y <- coordinates(kde_high_sp)[,2] 

l1 <- bbox(kde_high_sp)[1,1] - (bbox(kde_high_sp)[1,1]*0.0001) 
l2 <- bbox(kde_high_sp)[1,2] + (bbox(kde_high_sp)[1,2]*0.0001)
l3 <- bbox(kde_high_sp)[2,1] - (bbox(kde_high_sp)[2,1]*0.0001)
l4 <- bbox(kde_high_sp)[2,2] + (bbox(kde_high_sp)[1,1]*0.0001)

library(spatstat)  
win <- owin(xrange=c(l1,l2), yrange=c(l3,l4)) # 지도 경계선 생성
plot(win)         
rm(list = c("kde_high_sp", "apt_price", "l1", "l2", "l3", "l4"))
```
1-4: 밀도 그래프 표시
```
p <- ppp(x, y, window=win)  
d <- density.ppp(p, weights=kde_high$avg_price, 
                 sigma = bw.diggle(p), 
                 kernel = 'gaussian')  
plot(d)   
rm(list = c("x", "y", "win","p"))
```
1-5: 래스터 이미지로 변환
```
d[d < quantile(d)[4] + (quantile(d)[4]*0.1)] <- NA  
library(raster)      
raster_high <- raster(d) 
plot(raster_high)
```
1-6: 불필요한부분 자르기(클리핑)
```
bnd <- st_read("./01_code/sigun_bnd/seoul.shp")    
raster_high <- crop(raster_high, extent(bnd))      
crs(raster_high) <- sp::CRS("+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 + towgs84=0,0,0") 
plot(raster_high)  
plot(bnd, col=NA, border = "red", add=TRUE)
```
1-7: 지도 그리기
```
library(rgdal)    
library(leaflet)  
leaflet() %>% 
  
  addProviderTiles(providers$CartoDB.Positron) %>% 
  addPolygons(data = bnd, weight = 3, color= "red", fill = NA) %>% 
  addRasterImage(raster_high, 
   colors = colorNumeric(c("blue", "green","yellow","red"), 
   values(raster_high), na.color = "transparent"), opacity = 0.4)
```
1-8: 저장,메모리 정리
```
dir.create("07_map")  
save(raster_high, file="./07_map/07_kde_high.rdata") 
rm(list = ls())
```

10월26일 9주차
==================
지오 데이터프레임 만들기

1.데이터 불러오기
* 주소 불러오기
* 좌표 불러오기
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
load("./04_preprocess/04_preprocess.rdata")    
load("./05_geocoding/05_juso_geocoding.rdata")
```
2. 주소 + 좌표 결합
* 결측치 제거
```
install.packages('dplyr')
library(dplyr)   
apt_price <- left_join(apt_price, juso_geocoding, 
                       by = c("juso_jibun" = "apt_juso"))
apt_price <- na.omit(apt_price)  
```

3.지오데이터프레임 생성
* 좌표값 할당
* 좌표게 정의
* sp형을 sf형으로 변환
```
install.packages('sp')
library(sp)   
coordinates(apt_price) <- ~coord_x + coord_y    
proj4string(apt_price) <- "+proj=longlat +datum=WGS84 +no_defs" 
install.packages('sf')
library(sf)     
apt_price <- st_as_sf(apt_price)    
```

4.지오데이터프레임 시각화
* 플롯 그리기
* 지도 그리기
* 일부분(1000개)만 그리기
```
plot(apt_price$geometry, axes = T, pch = 1)         
library(leaflet)   # install.packages('leaflet')   
leaflet() %>% 
  addTiles() %>% 
  addCircleMarkers(data=apt_price[1:1000,], label=~apt_nm) 
 ```

5.지오 데이터프레임 저장
* 새로운 폴더 생성
* rdata 저장
* csv 저장
```
dir.create("06_geodataframe")   
save(apt_price, file="./06_geodataframe/06_apt_price.rdata") 
write.csv(apt_price, "./06_geodataframe/06_apt_price.csv")   
```


10월12일 7주차
==================
카카오맵 API로 지오 코딩

1.지오 코딩 준비
* 카카오 로컬 API키 발급받기
* 중복된 주소 제거
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
load( "./04_preprocess/04_preprocess.rdata")            
apt_juso <- data.frame(apt_price$juso_jibun) 
apt_juso <- data.frame(apt_juso[!duplicated(apt_juso), ])  
head(apt_juso, 2)  
```
2.주소를 좌표로 변환하는 지오코딩
* 지오코딩 준비
```add_list <- list()   # 빈 리스트 생성
cnt <- 0             # 반복문 카운팅 초기값 설정
kakao_key = ""# 인증키
```
* 지오 코딩하기
라이브러리 설치
```
install.packages('httr')
install.packages('RJSONIO')
install.packages('data.table')
install.packages('dplyr')

library(httr)        
library(RJSONIO)    
library(data.table)  
library(dplyr)
```
* for반복문과 예외 처리, 주소요청, 위경도 정보추출, 정보저장, 진행 상황 알림 메세지 출력, for반복문과 예외 처리 종료

```
for(i in 1:nrow(apt_juso)){ 
  #---# 에러 예외처리 구문 시작
  tryCatch(
    {
      #---# 주소로 좌표값 요청
      lon_lat <- GET(url = 'https://dapi.kakao.com/v2/local/search/address.json',
                     query = list(query = apt_juso[i,]),
                     add_headers(Authorization = paste0("KakaoAK ", kakao_key)))
      #---# 위경도만 추출하여 저장
      coordxy <- lon_lat %>% content(as = 'text') %>% RJSONIO::fromJSON()
      #---# 반복횟수 카운팅
      cnt = cnt + 1
      #---# 주소, 경도, 위도 정보를 리스트로 저장
      add_list[[cnt]] <- data.table(apt_juso = apt_juso[i,], 
        coord_x = coordxy$documents[[1]]$x, 
        coord_y = coordxy$documents[[1]]$y)
      #---# 진행상황 알림 메시지
      message <- paste0("[", i,"/",nrow(apt_juso),"] 번째 (", 
       round(i/nrow(apt_juso)*100,2)," %) [", apt_juso[i,] ,"] 지오코딩 중입니다: 
       X= ", add_list[[cnt]]$coord_x, " / Y= ", add_list[[cnt]]$coord_y)
      cat(message, "\n\n")
      #---# 예외처리 구문 종료
    }, error=function(e){cat("ERROR :",conditionMessage(e), "\n")}
  )
}
```
3.지오코딩 결과 저장
* 코드
```
juso_geocoding <- rbindlist(add_list)   # 리스트를 데이터프레임 변환
juso_geocoding$coord_x <- as.numeric(juso_geocoding$coord_x) # 좌표값 숫자형 변환
juso_geocoding$coord_y <- as.numeric(juso_geocoding$coord_y)
juso_geocoding <- na.omit(juso_geocoding)   # 결측치 제거
dir.create("./05_geocoding")   # 새로운 폴더 생성
save(juso_geocoding, file="./05_geocoding/05_juso_geocoding.rdata") # 저장
write.csv(juso_geocoding, "./05_geocoding/05_juso_geocoding.csv")
```


10월5일 6주차
==================
1.CSV파일 통합
* 앞에서 만든 CSV파일 300개를 하나로 합치는 작업
* CSV파일을 읽어서 파일에 목록으로 생성후 apt_price에 저장
* 작업 폴더 설정 -> 폴더 내 모든 파일명 읽기 -> 결합 -> 확인
* 코드
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
files <- dir("./02_raw_data")
library(plyr)
apt_price <-
  ldply(as.list(paste0("./02_raw_data/", files)), read.csv)
tail(apt_price, 2)
```

2.통합 데이터 저장
* 새로운 폴더를 생성하고 save()와 write.csv()로 데이터를 RDATA와 CSV형식으로 저장
* 코드
```
dir.create("./03_integrated")
save(apt_price, file = "./03_integrated/03_apt_price.rdata")
write.csv(apt_price, "./03_integrated/03_apt_price.csv")
```

1.수집한 데이터 불러오기
* 코드
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
options(warn = -1)

load("./03_integrated/03_apt_price.rdata")
head(apt_price, 2)
```

2.결측값과 공백 제거
* 결측값은 NA(not available)로 표현
* NA로 된 데이터는 계산 불가
`table(is.na(apt_price))`
* 결측값 확인 후 제거, 확인
```
apt_price <- na.omit(apt_price)
table(is.na(apt_price))
```
* 데이터 앞이나 뒤에 있는 빈 데이터를 공백 이라고 함 
`head(apt_price$price,2)`
* 문자열 데이터에서 공백을 제거하려면 stringr패키지가 제공하는 str_trim()함수를 사용
* 데이터프레임을 대상으로 공백을 제거하려면 apply()함수를 사용
```
library(stringr)
apt_price <- as.data.frame(apply(apt_price, 2, str_trim)) 
head(apt_price$price, 2)
```
3.매매 연월일 생성
* 설치
`install.packages("lubridate")`
`install.packages("dplyr")`
* 연월일, 연월, 자료 확인
```
library(lubridate)
library(dplyr)      
apt_price <- apt_price %>% mutate(ymd=make_date(year, month, day))  
apt_price$ym <- floor_date(apt_price$ymd, "month")                  
head(apt_price, 2)
```
4. 매매가 변환
* 문자형 데이터에는 큰따옴표가 표시되고, 숫자형 데이터에는 표시되지 않음
`head(apt_price$price, 3)`
* sub()함수는 stringr패키지에 들어있음
`apt_price$price <- apt_price$price %>% sub(",","",.) %>% as.numeric()
head(apt_price$price, 3)`

5.주소 조합
* 파생 변수 만들기
`head(apt_price$apt_nm, 30)`
* 괄호부터 시작하는 문자 삭제
* 아파트 이름 확인
`apt_price$apt_nm <- gsub("\\(.*","", apt_price$apt_nm) 
head(apt_price$apt_nm, 30)`
```
loc <- read.csv("./sigun_code.csv", fileEncoding="UTF-8")

apt_price <- merge(apt_price, loc, by = 'code')         
apt_price$juso_jibun <- paste0(apt_price$addr_2, " ", apt_price$dong," ",
                               apt_price$jibun," ",apt_price$apt_nm) 

head(apt_price, 2)
```
6.건축 연도,전용면적 변환
* 건축연도 확인
`head(apt_price$con_year, 3)`
* 건축 연도 숫자 변환
`apt_price$area <- apt_price$area %>% as.numeric() %>% round(0)`
* 전용 면적 확인
`head(apt_price$area, 3)`
* 전처리 작업
`apt_price$py <- round(((apt_price$price/apt_price$area) * 3.3), 0)
head(apt_price$py, 3)`

7.평당 매매가 만들기
* 평당 가격, 매매가 확인
`apt_price$py <- round(((apt_price$price/apt_price$area) * 3.3), 0)
head(apt_price$py, 3)`

8.층수 변환하기
* 최솟값을 찾는 함수: min()
`min(apt_price$floor)`
* 층수를 숫자값으로 바꾸고 절댓값 함수인 abs()로 모두 양수로 바꿔줌
`apt_price$floor <- apt_price$floor %>% as.numeric() %>% abs() 
min(apt_price$floor)`
* cnt <-1로 모든 거래 건수에 1이라는 숫자를 부여
`apt_price$cnt <- 1   
head(apt_price, 2)`


9월28일 5주차
==================
1.자료 요청하고 응답받기
* URL목록인 url_list를 xmlTreeParse()로 보내고,응답결과를 raw_data[[i]]에 저장
* xmlRoot()로 XML의 루트 노드만 추출하여 임시저장소인 root_Node[[i]]에 저장
* 코드
```
for(i in 1:length(url_list)){ 
  raw_data[[i]] <- xmlTreeParse(url_list[i], useInternalNodes = TRUE,encoding = "utf-8")
  root_Node[[i]] <- xmlRoot(raw_data[[i]])
```

2.전체 거래 건수 확인
* 전체 거래 내역 추출
`items <- root_Node[[i]][[2]][['items']]`
* 전체 거래 건수 확인
`size <- xmlSize(items)`

3.개별 거래 내역 추출
- 전체 거래 내역(items) 저장 임시 리스트를 생성
- 세부 거래 내역(items) 저장 임시 테이블을 생성
- 0.1초를 멈추게하고
- 거래 연도,월,일,금액,지역코드,법정동,지번,건축 연도,아파트이름,전용면적,층수를 분리된 거래 내역 순서대로 저장 후 통합 저장한다.
* 코드
```
  item <- list()  
  item_temp_dt <- data.table()
  Sys.sleep(.1)  
  for(m in 1:size){  
    item_temp <- xmlSApply(items[[m]],xmlValue)
    item_temp_dt <- data.table(year = item_temp[4],     
                               month = item_temp[7],    
                               day = item_temp[8],      
                               price = item_temp[1],    
                               code = item_temp[12],    
                               dong_nm = item_temp[5],  
                               jibun = item_temp[11],   
                               con_year = item_temp[3], 
                               apt_nm = item_temp[6],     
                               area = item_temp[9],     
                               floor = item_temp[13])   
    item[[m]] <- item_temp_dt}    
  apt_bind <- rbindlist(item)
```

4.응답 내역 저장
- subset()함수로 지역명 추출
- str_sub() = 문자열 추출 함수
- paste0()로 region_cd와 month를 조합한 저장위치에 path를 설정하고 write.csv()로 실거래 데이터를 저장
* 코드
```
region_nm <- subset(loc, code== str_sub(url_list[i],115, 119))$addr_1
month <- str_sub(url_list[i],130, 135)
path <- as.character(paste0("./02_raw_data/", region_nm, "_", month,".csv"))
write.csv(apt_bind, path)   
msg <- paste0("[", i,"/",length(url_list), "] 수집한 데이터를 [", path,"]에 저장 합니다.")
cat(msg, "\n\n")
}

```

9월21일 4주차
==================
1.요청 목록 만들기
* 빈 리스트 만들기
`url_list <- list()`
* 반복문의 제어 변수 초깃값 설정
`cnt <- 0`

2.요청목록 채우기
```
for(i in 1:nrow(loc)){
  for(j in 1:length(datelist)){
    cnt <- cnt + 1
    url_list[cnt] <- paste0("http://openapi.molit.go.kr:8081/OpenAPI_ToolInstallPackage/service/rest/RTMSOBJSvc/getRTMSDataSvcAptTrade?",
                            "LAWD_CD=", loc[i,1],
                            "&DEAL_YMD=", datelist[j],
                            "&numOfRows=",100,
                            "&serviceKey=", service_key)
  }
  Sys.sleep(0.1)
  msg <- paste0("[",i,"/",nrow(loc),"]",loc[i,3],"의 크롤링 목록이 생성됨 => 총[",cnt,"]건")
  cat(msg, "\n\n")
}
```
3.요청 목록 확인하기
* 요청목록 개수 확인
`length(url_list)`
*정상 동작 확인
`browseURL(paste0(url_list[1})`

4.임시 저장 리스트 생성
* 설치
```
install.packages("XML")
install.packages("data.table")
install.packages("stringr")

```
* XML 임시 저장소 
`raw_data <- list()`
* 거래 내역 추출 임시 저장소
`root_Node <- list()`
* 거래 내역 정리 임시 저장소
`total <- list()`
* 새로운 폴더 만들기
`dir.create("02_raw_data")`

9월14일 3주차
==================
1.작업폴더 설정
* rstudioapi설치
`install.packages("rstudioapi")`
* 작업 폴더 설정
`setwd(dirname(rstudioapi::getSourceEditorContext()$path))`
* 작업 폴더 확인
`getwd()`

2.수집대상 지역 설정
* 지역 코드
`loc <-read.csv("./sigun_code.csv", fileEncoding = "UTF-8")`
* 행정구역명 문자 변환
`loc$code < -as.character(loc$code)`
* 확인
`head(loc,2)`

3.수집기간 설정
```
datelist <- seq(from =as.Date('2021-01-01'),
                to = as.Date('2021-12-31'),
                by = '1 month')
datelist<- format(datelist, format = '%Y%m')
datelist[1:5]
```
4.인증키 입력
`service_key <- `


2주차 9월 7일
===============
```
install.packages("wordcloud")
library(wordcloud)

word <- c("인천광역시","강화군","웅진군")
frequency <- c(651,185,61)

wordcloud(word,frequency,colors=rainbow(length(word)))
```
