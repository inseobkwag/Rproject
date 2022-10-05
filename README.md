# 602277101 곽인섭

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
