# 602277101 곽인섭

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
