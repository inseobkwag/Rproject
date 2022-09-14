9월 14일
#rstudioapi설치
install.packages("rstudioapi")
#작업 폴더 설정
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
#작업 폴더 확인
getwd()

#수집대상 지역 설정
#지역 코드
loc <-read.csv("./sigun_code.csv", fileEncoding = "UTF-8")
#행정구역명 문자 변환
loc$code < -as.character(loc$code)
#확인
head(loc,2)

#수집기간 설정
datelist <- seq(from =as.Date('2021-01-01'),
                to = as.Date('2021-12-31'),
                by = '1 month')
datelist<- format(datelist, format = '%Y%m')
#확인
datelist[1:5]
9월 7일
# Rproject
```
install.packages("wordcloud")
library(wordcloud)

word <- c("인천광역시","강화군","웅진군")
frequency <- c(651,185,61)

wordcloud(word,frequency,colors=rainbow(length(word)))
```
