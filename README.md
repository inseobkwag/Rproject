# Rproject
```
install.packages("wordcloud")
library(wordcloud)

word <- c("인천광역시","강화군","웅진군")
frequency <- c(651,185,61)

wordcloud(word,frequency,colors=rainbow(length(word)))
```
