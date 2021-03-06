---
title: "Health and Economic Effects of Different Types of Severe Weather on United States (1950-2011)"
output: html_document
---

##Synopsis

We examined the health and economic problems created by different types of severe weather using data from 1950 to November 2011 collected in the United States by the NOAA.

Health effects were defined as direct injuries and deaths resulting from severe weather. Economic effects were defined as the value of direct damage to property and crops resulting from severe weather.

For the purposes of this exploratory analysis we considered the United States as a whole and examined the data across all available dates in the sample.

Based on our analysis:

 **Tornadoes cause the most harm to health**, in terms of both fatalities and injuries. 
 
 **Floods cause the most economic harm**, although droughts cause more damage to crops. 

##About the Data

The data for this project uses a comma-separated-value file compressed via the bzip2 algorithm. 

There is also some documentation of the database available. Here you will find how some of the variables are constructed/defined.

[Storm Data (47MB)](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2)
[National Weather Service Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf)

[National Climatic Data Center Storm Events FAQ](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf)

The events in the database start in the year 1950 and end in November 2011. In the earlier years of the database there are generally fewer events recorded, most likely due to a lack of good records. More recent years should be considered more complete.

##Data Processing

###Setup


```r
library('dplyr')
library('scales')
library('ggplot2')
library('tidyr')
library('knitr')

opts_chunk$set(echo=TRUE, results='hide', fig.path='figure/',tidy=TRUE)
```

No initial modification was made to the data besides converting it into a dplyr data frame for easier analysis.

###Raw Data


```r
if (!file.exists("StormData.csv.bz2")) {
    download.file("https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2", 
        "StormData.csv.bz2", method = "curl")
    downloadDate <- date()
} else {
    downloadDate <- format(file.info("StormData.csv.bz2")$mtime, "%a %b %d %H:%M:%S %Y")
}

if (!exists("rawStormData")) {
    rawStormData <- tbl_df(read.csv("StormData.csv.bz2"))
}
```

The data  was last accessed on Sun Apr 26 11:08:23 2015.

###Health Consequences

We first examined the health consequences of different types of severe weather. We took a subset of the data that included event type (EVTYPE), fatalities (FATALITIES), and injuries (INJURIES). We took the sum of fatalities and injuries by event type.

To focus on the most pressing types of severe weather we took only the results that ranked in the top one percent of total injuries or total fatalities. We then examined the top fatality and injury causing event types. 

For the purposes of this dataset fatalities and injuries refer to direct fatalities and injuries resulting from the event.

We examined the top 5 results for injuries and fatalities and then calculated the harm and percent of total injuries and fatalities that tornadoes were responsible for.

We plotted the two types of health damage and the event types.

Relevant tables and figures are located in *Results* under *Health Consequences*.


```r
populationEffect <- select(rawStormData, one_of(c("EVTYPE", "FATALITIES", "INJURIES"))) %>% 
    group_by(EVTYPE) %>% summarise(FATALITIES = sum(FATALITIES), INJURIES = sum(INJURIES))

top1PercentPopulation <- populationEffect %>% filter(percent_rank(FATALITIES) > 
    0.99 | percent_rank(INJURIES) > 0.99) %>% gather(RESULTTYPE, NUMBER, -EVTYPE) %>% 
    transform(EVTYPE = reorder(EVTYPE, NUMBER))

top5Fatalities <- select(populationEffect, EVTYPE, FATALITIES) %>% top_n(5, 
    FATALITIES) %>% arrange(desc(FATALITIES))

top5Injuries <- select(populationEffect, EVTYPE, INJURIES) %>% top_n(5, INJURIES) %>% 
    arrange(desc(INJURIES))

fatalitiesTornado <- subset(top1PercentPopulation, RESULTTYPE == "FATALITIES" & 
    EVTYPE == "TORNADO", select = NUMBER)
percentFatalitiesTornado <- fatalitiesTornado/sum(subset(top1PercentPopulation, 
    RESULTTYPE == "FATALITIES", select = NUMBER))

injuriesTornado <- subset(top1PercentPopulation, RESULTTYPE == "INJURIES" & 
    EVTYPE == "TORNADO", select = NUMBER)
percentInjuriesTornado <- injuriesTornado/sum(subset(top1PercentPopulation, 
    RESULTTYPE == "INJURIES", select = NUMBER))

barplotHealth <- ggplot(top1PercentPopulation, aes(x = EVTYPE, y = NUMBER)) + 
    geom_bar(aes(fill = RESULTTYPE), stat = "identity") + theme(axis.text.x = element_text(angle = 90, 
    hjust = 1)) + scale_fill_discrete(name = "Harm Type", labels = c("Fatalities", 
    "Injuries")) + xlab("Severe Weather Type") + ylab("Incidents of Harm") + 
    ggtitle("Health Effects of Severe Weath")
```

###Economic Consequences

We next examined the economic consequences of different types of severe weather. We took a subset of the data that included event type (EVTYPE), property damage (PROPDMG), and crop damage (CROPDMG), as well as their corresponding exponent (EXP) columns. 

We converted the alphanumeric exponents to numeric for known prefixes (H,K,M,B), all other exponents were converted to 1 so the corresponding crop damage could still be recorded. To get the actual crop and property damage we multiplied the number in that column by the exponent listed in the corresponding column i.e. CROPDMGEXP for CROPDMG. We took the sum of property and crop damage by event type.

For the purposes of this dataset damage refers to direct damage resulting from the event.

To focus on the most pressing types of severe weather we took only the results that ranked in the top one percent of total property damage or total crop damage. We then examined the top property and crop damage causing event types. 

For the purposes of this dataset property and crop damage refer to direct damage resulting from the event.

We examined the top 5 results for property and crop damage and then calculated the damage and percent of total property and crop damage that floods and droughts were responsible for.

We plotted the two types of economic damage and the event types. The economic damage was converted to use the unit "millions of dollars".

Relevant figures and plots are located in *Results* under *Economic Consequences*.


```r
economicEffect <- select(rawStormData, one_of(c("EVTYPE", "PROPDMG", "PROPDMGEXP", 
    "CROPDMG", "CROPDMGEXP"))) %>% mutate(CROPDMGEXP = ifelse(tolower(CROPDMGEXP) == 
    "b", 1e+09, ifelse(tolower(CROPDMGEXP) == "m", 1e+06, ifelse(tolower(CROPDMGEXP) == 
    "k", 1000, ifelse(tolower(CROPDMGEXP) == "h", 100, 1))))) %>% mutate(PROPDMGEXP = ifelse(tolower(PROPDMGEXP) == 
    "b", 1e+09, ifelse(tolower(PROPDMGEXP) == "m", 1e+06, ifelse(tolower(PROPDMGEXP) == 
    "k", 1000, ifelse(tolower(PROPDMGEXP) == "h", 100, 1))))) %>% mutate(PROPDMG = PROPDMG * 
    PROPDMGEXP, CROPDMG = CROPDMG * CROPDMGEXP) %>% group_by(EVTYPE) %>% summarise(PROPDMG = sum(PROPDMG), 
    CROPDMG = sum(CROPDMG))

top1PercentEconomic <- economicEffect %>% filter(percent_rank(CROPDMG) > 0.99 | 
    percent_rank(PROPDMG) > 0.99) %>% gather(RESULTTYPE, NUMBER, -EVTYPE) %>% 
    transform(EVTYPE = reorder(EVTYPE, NUMBER))

top5Crop <- select(economicEffect, EVTYPE, CROPDMG) %>% top_n(5, CROPDMG) %>% 
    arrange(desc(CROPDMG))

top5Property <- select(economicEffect, EVTYPE, PROPDMG) %>% top_n(5, PROPDMG) %>% 
    arrange(desc(PROPDMG))

propertyDamageFlood <- subset(top1PercentEconomic, RESULTTYPE == "PROPDMG" & 
    EVTYPE == "FLOOD", select = NUMBER)
percentPropertyDamageFlood <- propertyDamageFlood/sum(subset(top1PercentEconomic, 
    RESULTTYPE == "PROPDMG", select = NUMBER))

cropDamageDrought <- subset(top1PercentEconomic, RESULTTYPE == "CROPDMG" & EVTYPE == 
    "DROUGHT", select = NUMBER)
percentCropDamageDrought <- cropDamageDrought/sum(subset(top1PercentEconomic, 
    RESULTTYPE == "CROPDMG", select = NUMBER))

cropDamageFlood <- subset(top1PercentEconomic, RESULTTYPE == "CROPDMG" & EVTYPE == 
    "FLOOD", select = NUMBER)
percentCropDamageFlood <- cropDamageFlood/sum(subset(top1PercentEconomic, RESULTTYPE == 
    "CROPDMG", select = NUMBER))

formatterMillionDollar <- function(x) {
    dollar(x/1e+06)
}

barplotEconomic <- ggplot(top1PercentEconomic, aes(x = EVTYPE, y = NUMBER)) + 
    geom_bar(aes(fill = RESULTTYPE), stat = "identity") + theme(axis.text.x = element_text(angle = 90, 
    hjust = 1)) + scale_fill_discrete(name = "Damage Type", labels = c("Property", 
    "Crops")) + scale_y_continuous(labels = formatterMillionDollar) + xlab("Severe Weather Type") + 
    ylab("Economic Damage in Millions of $") + ggtitle("Economic Effects of Severe Weather")
```

##Results

###Health Consequences

**Tornadoes** were the type of severe weather that caused the greatest harm to the population. 

Tornadoes caused 91,346 injuries and 5,633 fatalities. 

Tornadoes were responsible for **72%** of injuries  and **46%** of fatalities caused by severe weather.

![plot of chunk plotHarmHealth](figure/plotHarmHealth-1.png) 

For additional detail, top five most damaging types of severe weather in terms of both fatalities and injuries are shown in the tables below.


|EVTYPE         | FATALITIES|
|:--------------|----------:|
|TORNADO        |       5633|
|EXCESSIVE HEAT |       1903|
|FLASH FLOOD    |        978|
|HEAT           |        937|
|LIGHTNING      |        816|




|EVTYPE         | INJURIES|
|:--------------|--------:|
|TORNADO        |    91346|
|TSTM WIND      |     6957|
|FLOOD          |     6789|
|EXCESSIVE HEAT |     6525|
|LIGHTNING      |     5230|



###Economic Consequences

**Floods** were the type of severe weather that caused the greatest economic harm. 

Floods caused $144,657,709,807 property damage and $5,661,968,450 crop damage. 

Floods were responsible for **37%** of property damage and **13%** of crop damage caused by severe weather. Droughts caused more crop damage (32% or $13,972,566,000) but less overall economic damage.

![plot of chunk plotHarmEconomic](figure/plotHarmEconomic-1.png) 

For additional detail, top five most damaging types of severe weather in terms of both property and crop damage are shown in the tables below.


|EVTYPE            |      PROPDMG|
|:-----------------|------------:|
|FLOOD             | 144657709807|
|HURRICANE/TYPHOON |  69305840000|
|TORNADO           |  56937160779|
|STORM SURGE       |  43323536000|
|FLASH FLOOD       |  16140812067|




|EVTYPE      |     CROPDMG|
|:-----------|-----------:|
|DROUGHT     | 13972566000|
|FLOOD       |  5661968450|
|RIVER FLOOD |  5029459000|
|ICE STORM   |  5022113500|
|HAIL        |  3025954473|


