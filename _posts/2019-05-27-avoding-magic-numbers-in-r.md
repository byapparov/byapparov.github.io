## Turning magic numbers into user input

When you are working on a research you should try to avoid
[magic numbers](https://en.wikipedia.org/wiki/Magic_number_(programming)) as one
of [code smells] that makes the code difficult to reuse.

Most common example of this is usage of the date ranges that are mentioned in
different places of the code as filters:

```R
# Before that other data is extracted with same or other filters applied
dt <- dbExectute(
  "SELECT * FROM sales
  WHERE
    date BETWEEN '2018-01-01' AND '2018-12-31'"
)
# After that goes the analysis
hist(dt$amount, breaks = bins)
# ...
```

Imagine at some point you decide to turn your analysis into a Shiny application,
it will be very difficult to do so reliably as the meaning of the dates could be
lost by that time and reverse engineering is would be a waste of time.

Obvious solution is to give these variables a name.

Inspired by Shiny's input system I have started to crate `input` object at the top of all my scripts.

Compare this example to the one above:

```R
input <- list(
  date_range = list(
    start_date = "2018-01-01",
    end_date = "2018-12-31"
  )
)

sql.template <- "SELECT * FROM sales
        WHERE
          date BETWEEN '%1$s' AND '%2$s'"
sql <- sprintf(sql, input$date_range[1], input$date_range[2])
dt <- dbExectute(sql)
hist(dt$amount, breaks = bins)
```

You can see that date range values with this change are documented and this script is easy to reuse.

Now lets see what needs to change to migrate this code to Shiny:

* you need to add `ui` part of the app which will make it interactive for the user.
* `input` object is now managed by the framework but access to the values is exactly the same.

```R

ui <- fluidPage(
  # You need to give users control over the inputs
  # to make your data tool interactive through
  # Shiny framework.
  dateRangeInput(
     "daterange",
     "Date range:",
     start = "2018-01-01",
     end   = "2018-12-31"
  ),
  mainPanel(
     # Output: histogram of sale amounts
     plotOutput(outputId = "distPlot")
  )
)

server <- function(input, output) {
  output$distPlot <- renderPlot({
    # Nothing has changed in this part of the code
    sql.template <- "SELECT * FROM sales
            WHERE
              date BETWEEN '%1$s' AND '%2$s'"
    sql <- sprintf(sql, input$date_range[1], input$date_range[2])
    dt <- dbExectute(sql)
    hist(dt$amount, breaks = bins)
  })
}
```

Minimum code changes turned ad hoc research script into an application that can be used by users independently.

It is just a little trick, but imagine if the whole team uses it consistently across the projects.
