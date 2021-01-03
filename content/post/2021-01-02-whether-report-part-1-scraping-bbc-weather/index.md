---
title: 'Whether Report? Part 1: Scraping BBC Weather'
author: ~
date: '2021-01-02'
slug: []
categories: []
tags: ["R", "statistics", "programming"]
---

# Weather forecasting forecasting

Last summer I was looking at the BBC's weather forecast for the coming two weeks, and it started to bug me that there were no error bars on these numbers. So I decided to add them for myself!

What I wanted to see was how the predictions got more and more reliable as the day approached. To see this for day *X*, I would need to start thirteen days earlier, on day *X -- 13*. I would record that temperature; then the temperature on day *X -- 12*, *X -- 11* and so on until day *X*. That would give me a table:

Days in the future | Predicted temperature | Actual temperature on day X | How far off?
-|-|-|-
13|24|21|3
12|23|21|2
11|22|21|1
10|23|21|2
9|22|21|1
8|20|21|-1
7|21|21|0
6|21|21|0
5|22|21|1
4|19|21|-2
3|20|21|-1
2|21|21|0
1|22|21|1
0|21|21|0 (by definition)

That's two weeks of predictions for the temperature on day *X*, which on day *X* itself ended up being 21 °C. The difference between the *predicted* temperature and the *actual* temperature could be graphed against how far in the future the prediction was made.

To get all the data I needed, I would have to check the BBC website each day, scrape all the temperatures, and store them in CSV format for later analysis together with certain information about when the data was collected.

# R

There is probably a smarter way of doing this, but for me this worked just fine. 

```{r, eval = FALSE}
# The weather report for my hown town
website <- "https://www.bbc.co.uk/weather/2640729"

# Where I want the data to end up
destination <- "~/Documents/R/whether/whether_data.csv"

# I'm going to save the entire HTML of the BBC's website to file. This is where I'll put it.
# I'm using paste0 to construct a unique filename based on the system date.
content_destination <- paste0("~/Documents/R/whether/content/content_", Sys.Date(), ".txt")

```

The BBC forecast site has 28 temperatures on it: 14 maximum temperatures, 14 minimum ones. Studying the website's HTML, I can see that these temperatures are each preceded by ```<span class=\"wr-value--temperature--c\">"```, so I can extract them using ```str_extract_all(x, pattern = "(?<=<span class=\"wr-value--temperature--c\">)-?\\d+")```. 

These will alternate between maximum and minumum predictions, so I'll assign them an alternating 0-1-0-1-0-1 flag to signify whether they're high or low.

There's a problem with this, however.

Normally I would scrape the website in the morning. However, if I (for some reason) scraped it in the evening, the maximum temperature won't be reported: the sequence of temperatures will start with the minimum, not the maximum, and my alternating pattern will be incorrect.

I prevended this from happening by only allowing my script to operate between the hours of 8 AM and 5 PM. I put the scaping code inside an if-block that checks the time, and also tests for the presence of a ```content_destination``` file to ensure that it only records one set of temperatures per day.

```{r, eval = FALSE}
# Checking whether it's allowed to scrape the website at this time
if(!file.exists(content_destination) &
   (as.integer(substr(Sys.time(), 12, 13)) <= 17) &
   (as.integer(substr(Sys.time(), 12, 13)) >= 8)) {
   
  # Load libraries
  library(tidyverse)
  library(RCurl)
  library(XML)
  
  
  # Download the HTML
  content <- RCurl::getURLContent(website)

  # Extract the temperatures
  temperatures <- 
    content %>%
    str_extract_all("(?<=<span class=\"wr-value--temperature--c\">)-?\\d+") %>%
    unlist()

  # Convert these into a table
  predictions <- 
    tibble(
      time_of_prediction = Sys.time(),    # When the website was scraped
      predicted_temperature_date = Sys.Date() + seq(0, 13.5, by = 0.5), # Which date is being predicted?
      predicted_temperature = temperatures[1:28],
      days_in_future = (predicted_temperature_date - Sys.Date()) %>% as.integer()
    )

  # Generating the alternating pattern of maximum (0) or minimum (1)
  predictions$low <- (seq(1:nrow(predictions)) + 1) %% 2

  # Write to file
  write_csv(predictions, path = destination, append = TRUE)

}

```

That's it! Every time this program is run it appends the day's data to the ```destination``` CSV. All I have to do is run the code every day.

# Automation

...which of course was never going to happen, because I would keep forgetting.

So I needed to automate the task. Fortunately I have a Raspberry Pi 4 sitting around doing not much.[^1] I could plug that in, connect it to the network, and just leave it on for six months hoovering up data once per day.

So I wrote a quick bash script:

```{bash}
#!/bin/bash

Rscript ~/Documents/R/whether/whether.R
```

I saved this as whether.sh, made it executable, then added it to my crontab to run every 15 minutes.

```
# crontab

*/15 * * * * ~/Documents/R/whether/whether.sh
```

Of course, scraping the BBC's website every 15 minutes would generate a lot of redundant data; that's why it's useful to have those checks in the R script to ensure the website is only scraped once per day, between the hours of 8AM and 5PM (so, in practice, precisely once per day at 8AM).

# Finshed!

All done! Now all I had to do is leave the Raspberry Pi plugged in, quietly downloading the daily weather forecast once per day.

The next part would be analysing the data. *To be continued...*

[^1]: The '4' is important. I'm using several functions from the tidyverse. Unfortunately, installing the tidyverse takes up a lot of memory (?) or something that models less powerful than the RPi4 don't have enough of. I could easily have rewritten this program not to use the tidyverse, but later scripts would automate graphing, and for that it the tidyverse would be essential anyway.