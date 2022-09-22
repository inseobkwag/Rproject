# 602277101 곽인섭

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
