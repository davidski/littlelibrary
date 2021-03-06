

```r
# Parse the PDF -----------------------------------------------------------

# The PDF is super messy. I couldn't get it to parse cleanly.

library(tm)
library(httr)

# uri <- "https://littlefreelibrary.org/wp-content/uploads/2013/08/World-Map-Appendix-Update-10.1.15.pdf", 
# if (!file.exists("data/librarymap.pdf")) {
#   download.file(url = uri, 
#                 destfile = "data/librarymap.pdf")
# }
# 
# uri <- sprintf("file://%s", file.path(getwd(), "data/librarymap.pdf"))
# pdf <- readPDF(engine="xpdf", control=list(text = "-table"))(elem=list(uri = uri), language="en")
# data <- pdf$content
# 
# col_widths <- c(8, (41-8), (52-41), (71-52), (84-71), (92-84), (149-92), (158-149), (167-158))
# col_names <- c("charter", "street_address", "city", "state", "postal", "library", "story", "lat", "long")
# dat <- read.fwf(textConnection(data), header=FALSE, fill=TRUE, row.names = NULL, 
#                 widths = col_widths, col.names = col_names)
# 

# Work with JSON ----------------------------------------------------------

# do a little magic with the mapping API to fetch all of the libraries in the US


library(jsonlite)
library(dplyr)
library(stringi)
library(jqr)

data_file <- "data/libraries2.json"

if (!file.exists(data_file)) {
  # save on hits/bandwidth to the littlelibrary map API
  
  uri <- "https://littlefreelibrary.secure.force.com/apexremote"
  # request params derived from Firefox developer tools and using copy-as-cURL
  header <- c(Host = "littlefreelibrary.secure.force.com", 
                 "User-Agent" = "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:47.0) Gecko/20100101 Firefox/47.0",
                 "X-User-Agent" = "Visualforce-Remoting",
                 Accept = "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
                 "Accept-Language" = "en-US,en;q=0.5",
                 DNT = "1",
                 "X-Requested-With" = "XMLHttpRequest",
                 "Referer" = "https://littlefreelibrary.secure.force.com/mapPage")
  post_data <- paste0('{"action":"MapPageController","method":"remoteSearch",',
                      '"data":["US","Country",null,null],"type":"rpc","tid":5,', 
                      '"ctx":{"csrf":"VmpFPSxNakF4Tmkwd05pMHlObFF4TmpveE1qb3dPQ', 
                      'zR4TnpCYSxqMWk5QlVlREZvX0dFSFMyckowN2Y1LFpUTmhZelkw",',
                      '"vid":"066d00000027Meh","ns":"","ver":29}}')
  cookie <- c(BrowserId = "KPy-yRlLREKvcrfwms0OZw")

  r <- POST(uri, add_headers(header), set_cookies(cookie), body = post_data, content_type_json())
  bin <- content(r, "raw")
  writeBin(bin, data_file)
}
```

```
## Warning in file(con, "wb"): cannot open file 'data/libraries2.json': No
## such file or directory
```

```
## Error in file(con, "wb"): cannot open the connection
```

```r
# use JQ to extract just the results and convert to a data.frame
dat <- jq(readLines(data_file, warn = FALSE), 
          '[.[0].result | .[].library]') %>% 
  fromJSON(., flatten = TRUE) 
```

```
## Warning in file(con, "r"): cannot open file 'data/libraries2.json': No such
## file or directory
```

```
## Error in file(con, "r"): cannot open the connection
```

```r
# clean up the names...why does the public map API return the email and exact location, 
# even if the steward has requested that those not be displayed?!
names(dat) <- c("library_name", "steward_name", "steward_email", 
                "charter_number", "exact_location", "email_display", 
                "name_display", "street", "city", "country", "postal_code", 
                "id", "library_story", "state", "lat", "long")

dat$country <- toupper(dat$country)
dat[dat$country == "USOFA","country"] <- "USA"
dat[dat$country == "US","country"] <- "USA"
dat <- filter(dat, country == "USA")

dat <- mutate(dat, state = toupper(state))

fix_state <- function(x) {
  # given a state name, try to convert it to a state abreviation
  if (x %in% state.abb) {
  } else {
    y <- stri_trans_totitle(x)
    if (y %in% state.name) {
      x <- state.abb[match(y, state.name)]
    }
  }
  x
}

# the state field is messy, apply some clean up logic
dat$state <- sapply(dat$state, fix_state)
dat$state <- gsub(".", "", dat$state, fixed = TRUE)
dat[dat$state == "ALL", "state"] <- "AL"
dat[dat$state == "WI WISCONSIN", "state"] <- "WI"
dat[dat$state == "MASSACUSETTS", "state"] <- "MA"
dat[dat$state == "NAPPANEE", "state"] <- "IN"

# Census Data -------------------------------------------------------------

# Fetch current population estimates from the US Census beurau

library(readr)
if (!file.exists("data/census.csv")) {
  download.file("http://www.census.gov/popest/data/national/totals/2015/files/NST-EST2015-alldata.csv",
                destfile = "data/census.csv")
}  
```

```
## Error in download.file("http://www.census.gov/popest/data/national/totals/2015/files/NST-EST2015-alldata.csv", : cannot open destfile 'data/census.csv', reason 'No such file or directory'
```

```r
census <- read_csv("data/census.csv") %>% filter(!is.na(NAME))
```

```
## Error: 'data/census.csv' does not exist in current working directory ('D:/littlelibrary/bin').
```

```r
census$state <- sapply(census$NAME, fix_state)
# population is in POPESTIMATE2015

# Join the data sets ------------------------------------------------------

libraries_by_state <- group_by(dat, state) %>% tally()
libraries_by_state <- left_join(libraries_by_state, select(census, state, population = POPESTIMATE2015))
```

```
## Error in UseMethod("tbl_vars"): no applicable method for 'tbl_vars' applied to an object of class "jqr"
```

```r
libraries_by_state %>% mutate(per_capita = n/population) -> libraries_by_state
```

```
## Error in eval(expr, envir, enclos): object 'population' not found
```

```r
# Plot the data -----------------------------------------------------------

library(ggplot2)
library(scales)
library(extrafont)

arrange(libraries_by_state, desc(per_capita)) %>% top_n(10) %>% 
  mutate(per_capita = per_capita) -> plot_data
```

```
## Error in eval(expr, envir, enclos): object 'per_capita' not found
```

```r
plot_data$state <- factor(plot_data$state, 
                          levels = plot_data[order(plot_data$per_capita, decreasing = FALSE), ]$state, 
                          ordered = TRUE)

gg <- ggplot(plot_data, aes(y = per_capita * 1000000, x = state, label = n))
gg <- gg + geom_segment(aes(y = 0, yend = per_capita * 1000000, x = state, 
                            xend = state), size = 7)
#gg <- gg + geom_text(hjust = 0, nudge_y = .001, size = 3)
gg <- gg + scale_y_continuous(labels = comma)
gg <- gg + labs(y = "Libraries per 1,000,000", x = "State", 
                title = "Free Lending Library Distribution", 
                subtitle = "Top 10 states with libraries per million population",
                caption = "Source: littlefreelibrary.org and\n2015 US Census population estimates")

# the total library count column
# column derived from technique demonstrated by @hrbrmstr
gg <- gg + geom_rect(data = plot_data, aes(ymin = 33, ymax = 38, 
                                           xmin = -Inf, xmax = Inf), 
                     fill = "#efefe3")
gg <- gg + geom_text(data = plot_data, aes(label = n, x = state, y = 34), 
                     fontface = "bold", size = 3, family = "Calibri")
gg <- gg + geom_text(data = filter(plot_data, state == "WI"), 
                     aes(y = 34, x = state, label = "Total #er"),
                     color = "#7a7d7e", size = 3.1, vjust = -2, fontface = "bold", 
                     family = "Calibri")
#gg <- gg + scale_x_continuous(expand=c(0,0), limits=c(0, 38))
#gg <- gg + scale_y_discrete(expand=c(0.075,0))

gg <- gg + coord_flip()
gg <- gg + theme_bw(base_family = "Calibri")
gg <- gg + theme(panel.grid.major = element_blank())
gg <- gg + theme(panel.grid.minor = element_blank())
gg <- gg + theme(panel.border = element_blank())
gg <- gg + theme(plot.title = element_text(face = "bold"))
gg <- gg + theme(plot.subtitle = element_text(face = "italic", size = 9, 
                                              margin = margin(b = 12)))
gg <- gg + theme(plot.caption = element_text(size = 7, margin = margin(t = 12), 
                                             color = "#7a7d7e"))
gg <- gg + theme_minimal()

gg
```

![plot of chunk unnamed-chunk-1](figure/unnamed-chunk-1-1.png)

```r
# Further Exploration on Seattle Metro Region -----------------------------

# zip code database from http://www.unitedstateszipcodes.org/zip-code-database/
# -- Couldn't get the downloads to work, so we wind up using the encoded
# city column
# zips <- read_csv("data/zip_code_database.csv")

seattle_zips <- c("98103", "98102")
dat %>% filter(postal_code == "98103") %>% View()

dat <- dat %>% mutate(city = str_to_title(city))

# Seattle is #3 in total number of libraries
dat %>% group_by(state, city) %>% tally() %>% View()

# One steward is associated with 19 libraries!
# Kimberly Daugherty, but they're all associated with a single address
dat %>% group_by(steward_name) %>% tally()  %>% arrange(desc(n)) %>% top_n(10)
```

```
## Selecting by n
```

```
## Source: local data frame [18 x 2]
## 
##               steward_name     n
##                      (chr) (int)
## 1       Kimberly Daugherty    19
## 2                 Deb West     6
## 3          Melissa Higgins     6
## 4              Katie Quinn     5
## 5               Gary Davis     4
## 6            Marie Johnson     4
## 7          Christine Barth     3
## 8          Barb Shillinger     2
## 9  Friends of Beckman Mill     2
## 10           J. A. Senesac     2
## 11            Kaye Johnson     2
## 12             Kim Carlson     2
## 13            Kolstad Rita     2
## 14            Laurie Marks     2
## 15             Maggie Vold     2
## 16          Patrick Hester     2
## 17     Priscilla Woolworth     2
## 18         Richard Venberg     2
```

```r
# There are 1142 unique charter numbers, but 1150 registered libraries
length(unique(dat$charter_number))
```

```
## [1] 1142
```

